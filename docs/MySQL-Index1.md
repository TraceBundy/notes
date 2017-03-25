#索引创建
在数据库中索引的作用很重要，那么如何创建一条好索引呢？下面从几个方面来说明，如何创建。

一般情况下，创建索引时通常都会用Explain来查看是否走了索引。但这样只能解决SQL是否走了索引，无法判断我们是否创建了一条高效的索引。
其实MySQL的sys库里给我们提高了有效的方法。
##sys.statistics 表
statistics表提供了索引的统计信息。
```
(root@localhost) [information_schema]>desc statistics;
+---------------+---------------+------+-----+---------+-------+
| Field         | Type          | Null | Key | Default | Extra |
+---------------+---------------+------+-----+---------+-------+
| TABLE_CATALOG | varchar(512)  | NO   |     |         |       |
| TABLE_SCHEMA  | varchar(64)   | NO   |     |         |       |
| TABLE_NAME    | varchar(64)   | NO   |     |         |       |
| NON_UNIQUE    | bigint(1)     | NO   |     | 0       |       |
| INDEX_SCHEMA  | varchar(64)   | NO   |     |         |       |
| INDEX_NAME    | varchar(64)   | NO   |     |         |       |
| SEQ_IN_INDEX  | bigint(2)     | NO   |     | 0       |       |
| COLUMN_NAME   | varchar(64)   | NO   |     |         |       |
| COLLATION     | varchar(1)    | YES  |     | NULL    |       |
| CARDINALITY   | bigint(21)    | YES  |     | NULL    |       |
| SUB_PART      | bigint(3)     | YES  |     | NULL    |       |
| PACKED        | varchar(10)   | YES  |     | NULL    |       |
| NULLABLE      | varchar(3)    | NO   |     |         |       |
| INDEX_TYPE    | varchar(16)   | NO   |     |         |       |
| COMMENT       | varchar(16)   | YES  |     | NULL    |       |
| INDEX_COMMENT | varchar(1024) | NO   |     |         |       |
+---------------+---------------+------+-----+---------+-------+
16 rows in set (0.00 sec)
```

从上面的表结构我们可以看到`CARDINALITY`字段，这个字段表示基数。它能够判断我们索引建立的好坏。
那我们怎么利用它来判断呢，这里有个公式
```
cardinality /table_rows  < 10% 选择度比较低的索引
```
我们举个例子:
student表结构
```
(root@localhost) [test]>desc student;
+-------+---------------+------+-----+---------+-------+
| Field | Type          | Null | Key | Default | Extra |
+-------+---------------+------+-----+---------+-------+
| name  | varchar(20)   | YES  |     | NULL    |       |
| sex   | enum('M','F') | YES  | MUL | NULL    |       |
| num   | int(11)       | YES  | UNI | NULL    |       |
+-------+---------------+------+-----+---------+-------+

我们给sex,num建立索引，其中num是唯一索引
```
表数据
```
(root@localhost) [test]>select * from student;
+-------+------+------+
| name  | sex  | num  |
+-------+------+------+
| yang  | M    |    1 |
| zhao  | F    |    2 |
| zhao  | M    |    3 |
| zhang | F    |    4 |
| zhang | F    |    5 |
| zhang | M    |    6 |
+-------+------+------+
```
我们看看statistics表的数据
```
(root@localhost) [test]>select * from information_schema.statistics where table_schema='test' and TABLE_NAME='student'\G;
*************************** 1. row ***************************
TABLE_CATALOG: def
 TABLE_SCHEMA: test
   TABLE_NAME: student
   NON_UNIQUE: 0
 INDEX_SCHEMA: test
   INDEX_NAME: u_num
 SEQ_IN_INDEX: 1
  COLUMN_NAME: num
    COLLATION: A
  CARDINALITY: 6
     SUB_PART: NULL
       PACKED: NULL
     NULLABLE: YES
   INDEX_TYPE: BTREE
      COMMENT:
INDEX_COMMENT:
*************************** 2. row ***************************
TABLE_CATALOG: def
 TABLE_SCHEMA: test
   TABLE_NAME: student
   NON_UNIQUE: 1
 INDEX_SCHEMA: test
   INDEX_NAME: i_sex
 SEQ_IN_INDEX: 1
  COLUMN_NAME: sex
    COLLATION: A
  CARDINALITY: 2
     SUB_PART: NULL
       PACKED: NULL
     NULLABLE: YES
   INDEX_TYPE: BTREE
      COMMENT:
INDEX_COMMENT:
2 rows in set (0.00 sec)
```

我们可以看到student的数据量是6， u_num的cardinality是6， i_sex的cardinality是2。
因为u_num索引是唯一索引，所以它的区分度是最高的 `100%`， i_sex是性别，只有两种情况，所以区分度会低 `33%`。

##判断复合索引的高效性
复合索引的高效性我们也可以通`statistics`表来看

```
*************************** 3. row ***************************
TABLE_CATALOG: def
 TABLE_SCHEMA: test
   TABLE_NAME: student
   NON_UNIQUE: 1
 INDEX_SCHEMA: test
   INDEX_NAME: i_num_name
 SEQ_IN_INDEX: 1
  COLUMN_NAME: num
    COLLATION: A
  CARDINALITY: 6
     SUB_PART: NULL
       PACKED: NULL
     NULLABLE: YES
   INDEX_TYPE: BTREE
      COMMENT:
INDEX_COMMENT:
*************************** 4. row ***************************
TABLE_CATALOG: def
 TABLE_SCHEMA: test
   TABLE_NAME: student
   NON_UNIQUE: 1
 INDEX_SCHEMA: test
   INDEX_NAME: i_num_name
 SEQ_IN_INDEX: 2
  COLUMN_NAME: name
    COLLATION: A
  CARDINALITY: 6
     SUB_PART: NULL
       PACKED: NULL
     NULLABLE: YES
   INDEX_TYPE: BTREE
      COMMENT:
INDEX_COMMENT:
```
SEQ_IN_INDEX 为1代表复合索引的第一个字段，SEQ_IN_INDEX 为2代表复合索引的前两个字段。一般我们认为`第一个字段选择对了就代表这个索引是高效的索引`
```
有（a,b)索引 是否要创建（a) 没有必要
```
##复合索引
复合索引(a,b)

```
 select * from t where a = ? 走索引
 select * from t where a = ? and b = ? 走索引
 select * from t where b = ? 不能
 select * from t where a = ? order by b 不需要文件排序
```
 对于文件排序，
 ```
 (root@localhost) [test]>desc select * from student where num > 1 order by name;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
|  1 | SIMPLE      | student | NULL       | ALL  | u_num         | NULL | NULL    | NULL |    6 |    83.33 | Using where; Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)
 ```
我们看到对name字段需要排序。那么怎么判断哪些SQL语句的文件排序多呢。
`sys`数据库中`statements_with_sorting`表的`rows_sorted`字段



##冗余索引
表中可能由于业务的修改，有些索引我们已经不再需要，或使用频率不是很高了。我们怎么能发现呢。`sys.schema_redundant_indexes`可以帮助我们

```
(root@localhost) [sys]>select * from schema_redundant_indexes where table_schema='test' limit 1\G ;
*************************** 1. row ***************************
              table_schema: test
                table_name: student
      redundant_index_name: i_num_name
   redundant_index_columns: num,name
redundant_index_non_unique: 1
       dominant_index_name: u_num
    dominant_index_columns: num
 dominant_index_non_unique: 0
            subpart_exists: 0
            sql_drop_index: ALTER TABLE `test`.`student` DROP INDEX `i_num_name`
```
不仅列出哪个索引，还给出删除索引的SQL

但这样粗暴的删除总是有隐患的，有可能这是最近没有用到。索引`MySQL8.0`中可以通过`alter table xxx alter index index_name invisible/visible `来将其隐藏。可以通过这种方式来观察数据库SQL运行情况，来判断是否要删除其索引。


##查看无用索引
`sys.schema_unused_indexes` 可以查看哪些是没有用到的索引。

##降序索引
`MySQL8.0`推出了降序索引，这个很有用
在`<=MySQL5.7`之前
```
select  * from test order by a desc , b
select * from test order by a, b desc
```
这些语句都有用到文件排序，不能走索引。
但`MySQL 8.0`可以通过
```
alter table orders add index index_name(a, b desc, c)
```
`MySQL5.7`可以通过虚拟列的方式`alter table tablename add column as (function(solumn)) virtual/stored`




