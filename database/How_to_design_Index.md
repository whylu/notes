# How to design Index
## Index PROS & CONS
- PROS
    - 快速找到資料
    - sort: index column 已經是排序過，可以節省排序時間

- CONS
    - slow on insert, update, delete


## How Index work


<br>

Index 結構 (出自 https://dataschool.com/sql-optimization/multicolumn-indexes/)

<img src="https://i.imgur.com/PTDaw5u.png" width="700px" />

```sql
CREATE TABLE TelephoneBook (
	last_name varchar(12) NOT NULL,
	first_name varchar(12) NOT NULL,
	phone_number varchar(12) NOT NULL
);
```

before create index
```sql
explain select * from TelephoneBook
where last_name = 'John'
-- results in:  Scan
Seq Scan on telephonebook  (cost=0.00..16.62 rows=3 width=126)
  Filter: ((last_name)::text = 'John'::text)
```



產生index 結構 last_name -> first_name -> phone_number 
```sql
CREATE INDEX phone_idx ON TelephoneBook(last_name, first_name, phone_number);
```

#### 使用 index 可以優化查詢效率
SELECT * FROM TelephoneBook WHERE: 
- (O) last_name = 'Smith'
    ```sql
    explain select * from TelephoneBook
    where last_name = 'Janessa'
    -- results in:  using index phone_idx
    Index Only Scan using phone_idx on telephonebook  (cost=0.28..4.29 rows=1 width=24)
      Index Cond: (last_name = 'Janessa'::text) -
    ```
- (O) last_name = 'Smith' AND first_name = 'John' 
    ```sql

    explain select * from TelephoneBook
    where last_name = 'John' and first_name ='Smith'
    -- results in:  jump last_name and first_name
    Index Only Scan using phone_idx on telephonebook  (cost=0.28..4.29 rows=1 width=24)
      Index Cond: ((last_name = 'John'::text) AND (first_name = 'Smith'::text))
    ```

- (O) last_name = 'John' and first_name ='Smith' and phone_number = '1234'
    ```sql
    explain select * from TelephoneBook
    where last_name = 'John' and first_name ='Smith' and phone_number = '1234'
    -- results in:  jump by last_name, first_name and phone_number
    Index Only Scan using phone_idx on telephonebook  (cost=0.28..4.30 rows=1 width=24)
      Index Cond: ((last_name = 'John'::text) AND (first_name = 'Smith'::text) AND (phone_number = '1234'::text))
    ```


- (O) last_name = 'John' and phone_number = '1234'
    ```sql
    explain select * from TelephoneBook
    where last_name = 'John' and phone_number = '1234'
    -- results in:  jump by last_name
    Index Only Scan using phone_idx on telephonebook  (cost=0.28..4.29 rows=1 width=24)
      Index Cond: ((last_name = 'John'::text) AND (phone_number = '1234'::text))
    ```

- (X) first_name = 'John' 
    ```sql
    explain select * from TelephoneBook
    where first_name = 'Chesley'
    -- results in: 
    Filter: ((first_name)::text = 'Chesley'::text)
    ```

#### 使用 index 時可以節略 sort 的操作:
```sql

-- ---- no index used
explain select * from TelephoneBook
where  first_name ='Smith'
order by last_name
-- results in: 
Sort  (cost=20.51..20.52 rows=1 width=24)
  Sort Key: last_name
  ->  Seq Scan on telephonebook  (cost=0.00..20.50 rows=1 width=24)
        Filter: ((first_name)::text = 'Smith'::text)
        
-- ## using index
explain select * from TelephoneBook
where  last_name ='Smith'
order by last_name
-- results in: 
Index Only Scan using phone_idx on telephonebook  (cost=0.28..4.29 rows=1 width=24)
  Index Cond: (last_name = 'Smith'::text)
        
```

#### index 支援 `like` ＊
```sql
explain select * from TelephoneBook
where  last_name like 'Jo%'
-- results in: 
Index Only Scan using phone_idx_pattern on telephonebook  (cost=0.28..4.50 rows=10 width=24)
  Index Cond: ((last_name ~>=~ 'Jo'::text) AND (last_name ~<~ 'Jp'::text))
  Filter: ((last_name)::text ~~ 'Jo%'::text)


explain select * from TelephoneBook
where  last_name like 'Jo%' and first_name like 'Smi%'
-- results in: 
Index Only Scan using phone_idx_pattern on telephonebook  (cost=0.28..4.44 rows=1 width=24)
  Index Cond: ((last_name ~>=~ 'Jo'::text) AND (last_name ~<~ 'Jp'::text) AND (first_name ~>=~ 'Smi'::text) AND (first_name ~<~ 'Smj'::text))
  Filter: (((last_name)::text ~~ 'Jo%'::text) AND ((first_name)::text ~~ 'Smi%'::text))  
```
but care for the order
```sql
explain  select * from TelephoneBook
where  last_name like 'Jo%' or first_name ='Carena'
-- index not used because first_name
Seq Scan on telephonebook  (cost=0.00..23.00 rows=11 width=24)
  Filter: (((last_name)::text ~~ 'Jo%'::text) OR ((first_name)::text = 'Carena'::text))
```

  - ＊ this is database specific, for [postgresql](https://www.postgresql.org/docs/9.4/indexes-opclass.html), need to using XXX_pattern_ops class
  ```sql
  CREATE INDEX phone_idx_pattern 
  ON TelephoneBook(last_name varchar_pattern_ops, first_name varchar_pattern_ops, phone_number varchar_pattern_ops) 
  ```

#### Covering Index
 如果 selecting-column 都已經包含在 index column 裡面，DB不需要執行讀取column value ，比如說
```sql
select last_name, phone_number from TelephoneBook
where last_name = 'John' and phone_number = '1234'
-- 使用了 phone_idx, index 裡面已經有 last_name, phone_number 資訊，DB 不必再去拿 last_name, phone_number


select last_name, contry from TelephoneBook
where last_name = 'John' and phone_number = '1234'
-- 使用了 phone_idx, index 裡面 沒有 contry 資訊，DB 必須再去拿 contry
```


## How to design Index of table

### Rate indexes by star 
如何評價一個 query 是有效率的使用 index 
- First-Star: index column 包含了所有的 query condition columns 
    ```sql
    where last_name = 'John' and phone_number = '1234'
    ```
- Second-Star: - query sort supported by the index
    ```sql
    order by last_name 
    ```
- Third-Star: index column 包含全部的 select-columns  (covering index)
    ```sql
    select last_name
    ```
#### First-Star is failed on:
-  `OR` condition
    ```sql
    explain select * from TelephoneBook
    where last_name = 'John' or first_name ='Carena'
    -- results in: 
    Seq Scan on telephonebook  (cost=0.00..23.00 rows=2 width=56)
      Filter: (((last_name)::text = 'John'::text) OR ((first_name)::text = 'Carena'::text))
    ```

#### Second-Star is failed on:
- range predicate, for example: `>`, `<`, `!=`, `NOT IN`, `BETWEEN`, `LIKE`, ...
    ```sql
  explain select * from TelephoneBook
  where  phone_number like '558%'
  order by phone_number
  -- results in: failed on sort supported by the index
  Sort  (cost=8.31..8.31 rows=1 width=56)
    Sort Key: phone_number
    ->  Index Scan using idx_phone_number on telephonebook  (cost=0.28..8.30 rows=1 width=56)
          Index Cond: (((phone_number)::text ~>=~ '558'::text) AND ((phone_number)::text ~<~ '559'::text))
          Filter: ((phone_number)::text ~~ '558%'::text)
    ```
- Query includes `ORDER BY` in a different order than the 'index order' 
    ```sql
  explain select last_name, first_name, phone_number from TelephoneBook
  where last_name = 'Ianilli' 
  order by last_name, first_name, phone_number -- this is in index order
  -- results in: 
  Index Only Scan using phone_idx on telephonebook  (cost=0.28..8.29 rows=1 width=24)
    Index Cond: (last_name = 'Ianilli'::text)

  -- But failed on 
  explain select last_name, first_name, phone_number   from TelephoneBook
  where last_name = 'Ianilli'  
  order by  last_name, phone_number, first_name
  -- results in: need a sort operation
  Sort  (cost=8.30..8.31 rows=1 width=24)
    Sort Key: phone_number, first_name
    ->  Index Only Scan using phone_idx on telephonebook  (cost=0.28..8.29 rows=1 width=24)
          Index Cond: (last_name = 'Ianilli'::text)
    ```
- Query includes `ORDER BY` in multiple columns over different tables.

#### Third-Star is failed on:
- Query more than maximum columns of a multicolumn-index can support.
- Query BLOB, TEXT, VARCHAR type longer than maximum bytes of index.



# REF
- https://www2.slideshare.net/billkarwin/how-to-design-indexes-really
- https://dataschool.com/sql-optimization/multicolumn-indexes/