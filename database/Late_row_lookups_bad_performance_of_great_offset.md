# late row lookups
面試幣安被問到一個問題
```sql
SELECT  *
FROM    news
ORDER BY
id DESC
LIMIT   150000, 10 
```
(which means limit 10 offset 150000 )  
這樣的 query 用 index 能不能改善效能


以下是優化的 query
```sql
SELECT  l.id, value, LENGTH(stuffing) AS len
FROM    (
	SELECT  id
	FROM    t_limit
	ORDER BY
	id
	LIMIT 150000, 10
) o
JOIN    t_limit l
ON      l.id = o.id
ORDER BY
l.id
```
原本的 query 會去抓其他欄位，才去做limit    
而sub-query 只抓出 id, 根據 Covering Index, 此時不用花時間去撈其他的欄位,   
但是要花額外的時間在 `JOIN    t_limit l`  
然而這些 join 花費 與 原本的花費比起來少了非常多

詳細可以看 [這篇文章](https://explainextended.com/2009/10/23/mysql-order-by-limit-performance-late-row-lookups/)
