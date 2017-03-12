##MySQL Select 语句

```
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
    select_expr [, select_expr ...]
    [FROM table_references
      [PARTITION partition_list]
    [WHERE where_condition]
    [GROUP BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [HAVING where_condition]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ...]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [FOR UPDATE | LOCK IN SHARE MODE]]

```
上面是SELECT SQL的常用部分。

逐一介绍：

*  `ALL` 默认即是
*  `DISTINCT` `DISTINCTROW` 去重相同的字段值

* `WHERE` 即是查询限制条件。
	
	**下面主要说一下 in、 not in、 exists之间的关心和问题。**
	```
	* IN equals =ANY
		SELECT A FROM t1 WHERE a IN (SELECT A FROM t2)
		SELECT a FROM t1 WHERE a = ANY (SELECT a from t2)
	* NOT IN 表示不在范围内
		注意 NOT IN 不走索引
	* EXISTS 
		* 仅返回TRUS 、FLASE
		* UNKNOWN返回为FALSE
		*  有时能和 IN互换
		SELECT uid, name FROM custoers AS A WHERE company = 'AMD' AND EXISTS (SELECT * FROM orders AS B WHERE A.uid = B.uid)
		SELECT uid, name FROM custoers AS A WHERE company = 'AMD' AND uid IN（SELECT uid FROM orders);
		
* GROUP BY 表示分组
	```
		例如订单，我们想求出每个月的总销售额 
		
			SELECT sum(orders_price) total , date_format(time, '%YY%m' FROM orders GROUP BY date_format(time, '%YY%m');
	```
	**GROUP BY要注意 在MySQL5.7中默认只能SELECT查询字段只能是聚合字段、或GROUP BY的分组字段。在MySQL5.6及以前是可以出现其它字段，返回的为随机的一条记录。但这是极力不推崇的。我们可以通过sql_mode=ONLY_FULL_GROUP_BY来使得5.6和5.7表现的一样**

	```
	GROUP BY 通常用到临时表。我们可以通过show status like'%tmp'来查看会话级别临时表的情况。如果太小则会影响到分组的速度。可以通过tmp_table_size来设置，默认是16M
	(root@localhost) [(none)]>show status like '%tmp%'
	-> ;
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 1     |
| Created_tmp_files       | 7     |
| Created_tmp_tables      | 1     |
+-------------------------+-------+
3 rows in set (0.01 sec)
	```

* HAVING 表示分组后的过滤，过滤是SELECT出现的列

* ORDER BY 表示排序，
	```
		我们可以通过sort_buffer_size来设置排序内存。
		通过show[global] status like '%sort%' 查看排序内存的情况。若Sort_merge_passes值很大，可以通过sort_buffer_size避免合并操作。
	```
* LIMIT 获取前多少条记录。
	```
		可以进行分页。单纯的LIMIT 分页在越往后性能越差。 
			1.可以通过每次查询待会上一次最后的记录作为下一次的查询条件来减少LIMIT的条数
				select id, name from user where id > last_id order by id limit 10
			2.可以通过一次性返回多条数据来减少网络次数。比如每页是10条，可以一次性返回30条
	```
	
####小贴士
```
count(1) count(*) 区别没有区别， 出来的就是行数
count(field)则统计的为非NULL的数量
```

```
(root@localhost) [(none)]>select NULL IN ('a', 'b', NULL);
+--------------------------+
| NULL IN ('a', 'b', NULL) |
+--------------------------+
|                     NULL |
+--------------------------+
1 row in set (0.00 sec)
(root@localhost) [dbt3]>select 'c'  in ('a','b', NULL);
+-------------------------+
| 'c'  in ('a','b', NULL) |
+-------------------------+
|                    NULL |
+-------------------------+
1 row in set (0.00 sec)

(root@localhost) [dbt3]>select 'a'  in ('a','b', NULL);
+-------------------------+
| 'a'  in ('a','b', NULL) |
+-------------------------+
|                       1 |
+-------------------------+
1 row in set (0.00 sec)
(root@localhost) [dbt3]>select 'a'  not in ('a','b', NULL);
+-----------------------------+
| 'a'  not in ('a','b', NULL) |
+-----------------------------+
|                           0 |
+-----------------------------+
1 row in set (0.00 sec)

带有NULL的字段，IN 用户返回 TRUE或 NULL, NOT IN 永远返回 NULL 或 FALSE 永远查不到数据 要过滤掉NULL值。
```
**所以不要在建表中使用NULL值，会引起很多莫名其妙的问题。**
```
SELECT 
	CONCAT(TABLE_SCHEMA,'.',TABLE_NAME) AS NAME,
    character_set_name,
    GROUP_CONCAT(COLUMN_NAME SEPARATOR ' : ') AS COLUMN_LIST
FROM information_schema.COLUMNS
WHERE
data_type IN ('varchar','longtext','text','mediumtext','char')
AND character_set_name <> 'utf8mb4'
AND table_schema NOT IN ('mysql' , 'performance_schema',
        'information_schema','sys')
GROUP BY NAME,character_set_name;
```





