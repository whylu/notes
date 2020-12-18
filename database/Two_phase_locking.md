# Two phase locking
To solve race condition on database, for example, double booking: two people booking the same seat at the same time.

## Example for double booking
With a table `seat`
```sql
CREATE TABLE seat (
	id serial PRIMARY KEY,
	username varchar(52)
);
```


| session for ming01 | session for peter0105 |
| ---- | ---- |
| begin transaction;<br>BEGINE | begin transaction;<br>BEGINE |
| UPDATE seat SET username = 'ming01' where id = 1;<br> UPDATE 1 | UPDATE seat SET username = 'peter0105' where id = 1;<br> (pending...) |
| COMMIT; | UPDATE 1 |
| SELECT * FROM seat WHERE id=1; <br>id \| username<br>1 \| ming01 | SELECT * FROM seat WHERE id=1; <br>id \| username<br>1 \| peter0105
| | COMMIT; |
| SELECT * FROM seat WHERE id=1; <br>id \| username<br>1 \| peter0105 | SELECT * FROM seat WHERE id=1; <br>id \| username<br>1 \| peter0105

The seat 1 for ming01 is overwrite by peter0105.


## Two phase locking protocol
- Phase 1: Expanding phase (aka Growing phase), acquired locks, no release allowed in this phase, the number of locks can only increase.
- Phase 2: Shrinking phase (aka Contracting phase), locks are released and no locks are acquired.

#### COMPATIBILITY MATRIX

| | Read lock | WRITE LOCK | 
|---- | ---- | ---- | 
| Read lock | Allow | Prevent |
| Write lock | Prevent| Prevent |


#### Clauses
| DATABASE NAME | READ LOCK CLAUSE | WRITE LOCK CLAUSE |
| ---- | ---- | ---- |
| PostgreSQL | FOR SHARE | FOR UPDATE |
| MySQL | LOCK IN SHARE MODE | FOR UPDATE |
check [explicit-locking](https://www.postgresql.org/docs/9.4/explicit-locking.html)



### Lock before update
booking a seat with lock

| session1 phase | session1 | session2 | session2 phase | 
| ---- | ---- | ---- | ---- | 
| Expanding | begin transaction;<br>BEGINE | begin transaction;<br>BEGINE | Expanding |
| Expanding | SELECT * FROM seat WHERE id = 2 AND username IS NOTNULL `FOR UPDATE`; <br>  id \| username <br>--- + ------- <br>2 \| <br> (1 row) (locked) | SELECT * FROM seat WHERE id = 2 AND username IS NOTNULL `FOR UPDATE`;<br> (pending...) <br> <br><br><br>| Expanding | 
| Expanding | UPDATE seat SET username = 'ming01' where id = 2; |  | Expanding | 
| Shrinking | COMMIT; | id \| username <br>--- + ------- <br> (0 row)  <br> (No seat id to book) | Expanding | 
| |  |  COMMIT; | Shrinking | 
| | SELECT * FROM seat WHERE id=1; <br>id \| username<br>1 \| ming01 |  SELECT * FROM seat WHERE id=1; <br>id \| username<br>1 \| ming01  |





# REF
- https://vladmihalcea.com/2pl-two-phase-locking/



