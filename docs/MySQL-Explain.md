
##Explain 例子
###Explain type
* `system` 只有一行记录的系统
* `const` 最多只有一行返回记录，如主键查询
* `eq_ref` 通过唯一键进行join
* `ref` 使用普通索引进行查询
* `fulltext` 使用全文索引进行查询
* `ref_or_null` 和 `ref` 类似，使用普通索引进行查询，但要查询NULL值
* `index_merge` or查询会使用到的类型
* `unique_subquery` 子查询的列是唯一索引
* `index_subquery` 子查询的列是普通索引
* `range` 范围扫描
* `index` 索引扫描
* `ALL` 全表扫描
###Explain Extra
* `using filesort` 使用额外的排序得到结果
* `using index` 优化器只需要使用索引就能得到结果
* `using index condition` 优化器使用index condition pushdown 优化
* `using index for group by` 优化器只需要使用索引就能处理group by 或distinct
* `using join buffer` 优化器需要使用join buffer join_buffer_size
* `using mrr` 优化器使用MRR优化
* `using temporary` 优化器需要使用临时表
* `using where` 优化器使用wher过滤

下面的例子大概常见的复杂例子啦

```
(root@localhost) [dbt3]>explain SELECT      * FROM     part WHERE     p_partkey IN (SELECT              l_partkey         FROM             lineitem         WHERE             l_shipdate BETWEEN '1997-01-01' AND '1997-02-01') ORDER BY p_retailprice DESC LIMIT 10;
+----+--------------+-------------+------------+--------+----------------------------------------------+--------------+---------+---------------------+--------+----------+----------------------------------+
| id | select_type  | table       | partitions | type   | possible_keys                                | key          | key_len | ref                 | rows   | filtered | Extra                            |
+----+--------------+-------------+------------+--------+----------------------------------------------+--------------+---------+---------------------+--------+----------+----------------------------------+
|  1 | SIMPLE       | part        | NULL       | ALL    | PRIMARY                                      | NULL         | NULL    | NULL                | 198104 |   100.00 | Using where; Using filesort      |
|  1 | SIMPLE       | <subquery2> | NULL       | eq_ref | <auto_key>                                   | <auto_key>   | 5       | dbt3.part.p_partkey |      1 |   100.00 | NULL                             |
|  2 | MATERIALIZED | lineitem    | NULL       | range  | i_l_shipdate,i_l_suppkey_partkey,i_l_partkey | i_l_shipdate | 4       | NULL                | 151632 |   100.00 | Using index condition; Using MRR |
+----+--------------+-------------+------------+--------+----------------------------------------------+--------------+---------+---------------------+--------+----------+----------------------------------+
3 rows in set, 1 warning (0.00 sec)
1 part 外表  subquery2产生派生表

内表要加索引 添加了一个唯一索引， 产生了一个实际的表 并建立一个唯一索引（in去重的 所以一定是唯一索引
```





```
(root@localhost) [dbt3]>explain SELECT      * FROM     part WHERE     p_partkey IN (SELECT              l_partkey         FROM             lineitem         WHERE             l_shipdate BETWEEN '1997-01-01' AND '1997-01-07') ORDER BY p_retailprice DESC LIMIT 10;
+----+--------------+-------------+------------+--------+----------------------------------------------+--------------+---------+-----------------------+-------+----------+----------------------------------------------+
| id | select_type  | table       | partitions | type   | possible_keys                                | key          | key_len | ref                   | rows  | filtered | Extra                                        |
+----+--------------+-------------+------------+--------+----------------------------------------------+--------------+---------+-----------------------+-------+----------+----------------------------------------------+
|  1 | SIMPLE       | <subquery2> | NULL       | ALL    | NULL                                         | NULL         | NULL    | NULL                  |  NULL |   100.00 | Using where; Using temporary; Using filesort |
|  1 | SIMPLE       | part        | NULL       | eq_ref | PRIMARY                                      | PRIMARY      | 4       | <subquery2>.l_partkey |     1 |   100.00 | NULL                                         |
|  2 | MATERIALIZED | lineitem    | NULL       | range  | i_l_shipdate,i_l_suppkey_partkey,i_l_partkey | i_l_shipdate | 4       | NULL                  | 32540 |   100.00 | Using index condition; Using MRR             |
+----+--------------+-------------+------------+--------+----------------------------------------------+--------------+---------+-----------------------+-------+----------+----------------------------------------------+



```
``` SQL
(root@localhost) [dbt3]>explain select max(l_extendedprice) from orders, lineitem where o_orderdate between '1995-01-01' and '1995-01-31' and l_orderkey = o_orderkey;
+----+-------------+----------+------------+-------+--------------------------------------------+---------------+---------+------------------------+-------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys                              | key           | key_len | ref                    | rows  | filtered | Extra                    |
+----+-------------+----------+------------+-------+--------------------------------------------+---------------+---------+------------------------+-------+----------+--------------------------+
|  1 | SIMPLE      | orders   | NULL       | range | PRIMARY,i_o_orderdate                      | i_o_orderdate | 4       | NULL                   | 40696 |   100.00 | Using where; Using index |
|  1 | SIMPLE      | lineitem | NULL       | ref   | PRIMARY,i_l_orderkey,i_l_orderkey_quantity | PRIMARY       | 4       | dbt3.orders.o_orderkey |     4 |   100.00 | NULL                     |
+----+-------------+----------+------------+-------+--------------------------------------------+---------------+---------+------------------------+-------+----------+--------------------------+
2 rows in set, 1 warning (0.00 sec)
```


```
(root@localhost) [dbt3]>explain select * from lineitem where l_shipdate <= '1995-01-31' union select * from lineitem where l_shipdate <= '1997-01-31'
    -> ;
+----+--------------+------------+------------+------+---------------+------+---------+------+---------+----------+-----------------+
| id | select_type  | table      | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra           |
+----+--------------+------------+------------+------+---------------+------+---------+------+---------+----------+-----------------+
|  1 | PRIMARY      | lineitem   | NULL       | ALL  | i_l_shipdate  | NULL | NULL    | NULL | 5858076 |    50.00 | Using where     |
|  2 | UNION        | lineitem   | NULL       | ALL  | i_l_shipdate  | NULL | NULL    | NULL | 5858076 |    50.00 | Using where     |
| NULL | UNION RESULT | <union1,2> | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    NULL |     NULL | Using temporary |
+----+--------------+------------+------------+------+---------------+------+---------+------+---------+----------+-----------------+

union 去重
```
```
(root@localhost) [employees]>explain select emp_no, dept_no, (select count(1) from dept_emp t2 where t1.emp_no <= t2.emp_no) as row_num from dept_emp t1;
+----+--------------------+-------+------------+-------+----------------+--------+---------+------+--------+----------+------------------------------------------------+
| id | select_type        | table | partitions | type  | possible_keys  | key    | key_len | ref  | rows   | filtered | Extra                                          |
+----+--------------------+-------+------------+-------+----------------+--------+---------+------+--------+----------+------------------------------------------------+
|  1 | PRIMARY            | t1    | NULL       | index | NULL           | emp_no | 4       | NULL | 331570 |   100.00 | Using index                                    |
|  2 | DEPENDENT SUBQUERY | t2    | NULL       | ALL   | PRIMARY,emp_no | NULL   | NULL    | NULL | 331570 |    33.33 | Range checked for each record (index map: 0x3) |
+----+--------------------+-------+------------+-------+----------------+--------+---------+------+--------+----------+------------------------------------------------+
2 rows in set, 2 warnings (0.00 sec)
这条语句就是先看1 在看2
```


```
(root@localhost) [dbt3]>explain SELECT     s_name,s_address FROM supplier,nation WHERE s_suppkey IN ( SELECT DISTINCT (ps_suppkey) FROM     partsupp,     part WHERE     ps_partkey=p_partkey     AND p_name LIKE 'orchid%'     AND ps_availqty > (     SELECT         0.5 * SUM(l_quantity)     FROM     lineitem     WHERE         l_partkey = ps_partkey         AND l_suppkey = ps_suppkey         AND l_shipdate >= '1996-01-01'         AND l_shipdate < DATE_ADD('1996-01-01',INTERVAL 1 YEAR)     ) ) AND s_nationkey = n_nationkey AND n_name = 'ALGERIA' ORDER BY s_name;
+----+--------------------+-------------+------------+--------+----------------------------------------------------------+---------------------+---------+---------------------------------------------------+--------+----------+----------------------------------------------+
| id | select_type        | table       | partitions | type   | possible_keys                                            | key                 | key_len | ref                                               | rows   | filtered | Extra                                        |
+----+--------------------+-------------+------------+--------+----------------------------------------------------------+---------------------+---------+---------------------------------------------------+--------+----------+----------------------------------------------+
|  1 | PRIMARY            | nation      | NULL       | ALL    | PRIMARY                                                  | NULL                | NULL    | NULL                                              |     25 |    10.00 | Using where; Using temporary; Using filesort |
|  1 | PRIMARY            | supplier    | NULL       | ref    | PRIMARY,i_s_nationkey                                    | i_s_nationkey       | 5       | dbt3.nation.n_nationkey                           |    398 |   100.00 | Using index condition                        |
|  1 | PRIMARY            | <subquery2> | NULL       | eq_ref | <auto_key>                                               | <auto_key>          | 4       | dbt3.supplier.s_suppkey                           |      1 |   100.00 | NULL                                         |
|  2 | MATERIALIZED       | part        | NULL       | ALL    | PRIMARY                                                  | NULL                | NULL    | NULL                                              | 198104 |    11.11 | Using where                                  |
|  2 | MATERIALIZED       | partsupp    | NULL       | ref    | PRIMARY,i_ps_partkey,i_ps_suppkey                        | PRIMARY             | 4       | dbt3.part.p_partkey                               |      3 |   100.00 | Using where                                  |
|  3 | DEPENDENT SUBQUERY | lineitem    | NULL       | ref    | i_l_shipdate,i_l_suppkey_partkey,i_l_partkey,i_l_suppkey | i_l_suppkey_partkey | 10      | dbt3.partsupp.ps_partkey,dbt3.partsupp.ps_suppkey |      7 |    28.04 | Using where                                  |
+----+--------------------+-------------+------------+--------+----------------------------------------------------------+---------------------+---------+---------------------------------------------------+--------+----------+----------------------------------------------+
6 rows in set, 3 warnings (0.00 sec)
```


####Tips
```

update x set a = a+1, b = a+10 where a = 2; 这样更新不是原子的
insert into z values (2) on duplicate key update a = a + 10
```
```
(root@localhost) [test]>select * from x;
+------+------+----+
| a    | b    | z  |
+------+------+----+
|   10 |   10 | 10 |
|   10 |   10 | 11 |
+------+------+----+
2 rows in set (0.00 sec)
alter table x add unique ss(z);
(root@localhost) [test]>replace into x values (10,19,11);
Query OK, 2 rows affected (0.00 sec)

(root@localhost) [test]>select * from x;
+------+------+----+
| a    | b    | z  |
+------+------+----+
|   10 |   10 | 10 |
|   10 |   19 | 11 |
+------+------+----+
2 rows in set (0.00 sec)
先删除重复key 再插入 所以是两条影响
在多个唯一索引时候 会有问题 数据会变少

replace 常用在复制 来实现幂等性
```
```
ibtmp1 5.7初始化会有 临时表空间。show variables like '%tmp%';可以看到 /tmp 中 会有临时表结构。 表的内容在ibtmp1中
+----------------------------------+----------+
| Variable_name                    | Value    |
+----------------------------------+----------+
| default_tmp_storage_engine       | InnoDB   |
| innodb_tmpdir                    |          |
| internal_tmp_disk_storage_engine | InnoDB   |
| max_tmp_tables                   | 32       |
| slave_load_tmpdir                | /tmp     |
| tmp_table_size                   | 16777216 |
| tmpdir                           | /tmp     |
+----------------------------------+----------+


```


