##InnoDB存储引擎缓存池
* InnoDB Buffer Pool
	* --innodb_buffer_poll_size
	* 越大越好 60%~70% 太大会容易导致OM问题

* BufferPool存储的是`(space,page_no)`

page_no|1|2|3|4
----|----|----|----|----
page|16K|16K|16K|16K
假设表空间`test.ibd`的表空间是`4`， 那么存储第一页即为`(4,0)`
space 可以通过 `INNODB_SYS_TABLESPACES`查看

```
(root@127.0.0.1) [information_schema]>select * from INNODB_SYS_TABLESPACES limit 5;
+-------+---------------------+------+-------------+------------+-----------+---------------+------------+---------------+-----------+----------------+
| SPACE | NAME                | FLAG | FILE_FORMAT | ROW_FORMAT | PAGE_SIZE | ZIP_PAGE_SIZE | SPACE_TYPE | FS_BLOCK_SIZE | FILE_SIZE | ALLOCATED_SIZE |
+-------+---------------------+------+-------------+------------+-----------+---------------+------------+---------------+-----------+----------------+
|     2 | mysql/plugin        |   33 | Barracuda   | Dynamic    |     16384 |             0 | Single     |          4096 |     98304 |          98304 |
|     3 | mysql/servers       |   33 | Barracuda   | Dynamic    |     16384 |             0 | Single     |          4096 |     98304 |          98304 |
|     4 | mysql/help_topic    |   33 | Barracuda   | Dynamic    |     16384 |             0 | Single     |          4096 |   9437184 |        9441280 |
|     5 | mysql/help_category |   33 | Barracuda   | Dynamic    |     16384 |             0 | Single     |          4096 |    114688 |         114688 |
|     6 | mysql/help_relation |   33 | Barracuda   | Dynamic    |     16384 |             0 | Single     |          4096 |    131072 |         131072 |
+-------+---------------------+------+-------------+------------+-----------+---------------+------------+---------------+-----------+----------------+
5 rows in set (0.00 sec)
```
* 通过 `INNODB_BUFFER_PAGE` 可以看到缓存池中存放的页

```
(root@127.0.0.1) [information_schema]>select * from INNODB_BUFFER_PAGE limit 5;
+---------+----------+-------+-------------+-----------+------------+-----------+-----------+---------------------+---------------------+-------------+---------------------+------------+----------------+-----------+-----------------+------------+---------+--------+-----------------+
| POOL_ID | BLOCK_ID | SPACE | PAGE_NUMBER | PAGE_TYPE | FLUSH_TYPE | FIX_COUNT | IS_HASHED | NEWEST_MODIFICATION | OLDEST_MODIFICATION | ACCESS_TIME | TABLE_NAME          | INDEX_NAME | NUMBER_RECORDS | DATA_SIZE | COMPRESSED_SIZE | PAGE_STATE | IO_FIX  | IS_OLD | FREE_PAGE_CLOCK |
+---------+----------+-------+-------------+-----------+------------+-----------+-----------+---------------------+---------------------+-------------+---------------------+------------+----------------+-----------+-----------------+------------+---------+--------+-----------------+
|       0 |        0 |   145 |         878 | INDEX     |          1 |         0 | YES       |          9907451731 |                   0 |  1131062091 | `sbtest`.`sbtest8`  | PRIMARY    |             73 |     15038 |               0 | FILE_PAGE  | IO_NONE | NO     |         2113386 |
|       0 |        1 |   143 |        1131 | INDEX     |          1 |         0 | YES       |          9862206950 |                   0 |  1131057890 | `sbtest`.`sbtest6`  | PRIMARY    |             73 |     15038 |               0 | FILE_PAGE  | IO_NONE | YES    |         2110800 |
|       0 |        2 |   144 |         601 | INDEX     |          1 |         0 | YES       |          9877503868 |                   0 |  1131059434 | `sbtest`.`sbtest7`  | PRIMARY    |             73 |     15038 |               0 | FILE_PAGE  | IO_NONE | YES    |         2111313 |
|       0 |        3 |   147 |        1474 | INDEX     |          2 |         0 | NO        |          9966784772 |                   0 |  1131067627 | `sbtest`.`sbtest10` | k_10       |           1203 |     15639 |               0 | FILE_PAGE  | IO_NONE | NO     |         2117114 |
|       0 |        4 |   143 |         714 | INDEX     |          1 |         0 | YES       |          9854641284 |                   0 |  1131057348 | `sbtest`.`sbtest6`  | PRIMARY    |             73 |     15038 |               0 | FILE_PAGE  | IO_NONE | YES    |         2110038 |
+---------+----------+-------+-------------+-----------+------------+-----------+-----------+---------------------+---------------------+-------------+---------------------+------------+----------------+-----------+-----------------+------------+---------+--------+-----------------+
5 rows in set (0.06 sec)
```
* 缓冲池存放的内容
	* 数据页
	* 索引页
	* change buffer
	* 自适应哈希
	* 锁(MySQL 5.5之前, 现在直接用malloc分配)
	
	##innodb_buffer_pool_instances 设置为CPU核数
	
* Buffer Pool
	* Free List
	* LRU List
		* LRU
		* unzip_LRU
	* Flush List
		根据oldest_lsn进行排序
		
		```
		show engine innodb status;
		----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 1990606
Buffer pool size   8191 缓冲池大小
Free buffers       1024 空闲页大小
Database pages     7159 LRU的数量
Old database pages 2622
Modified db pages  0 脏页大小
Pending reads      0
		```
##LRU管理
* 最近最少使用算法
* midpoint LRU 当从Free List 并不是放到最前面，而是放到3/8的位置
	
	new | midpoint| old
	----|----|----
	
	* 3/8
	* --innodb_old_blocks_pct={37}
	* --innodb_old_blocks_time 在midpoint位置保持多长时间，即使被读到多次也不移动到前面。在全表扫描时候适合使用
		* 避免扫描语句污染LRU
		
		```
		root@127.0.0.1) [test]>show global variables like '%innodb_old%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_old_blocks_pct  | 37    |
| innodb_old_blocks_time | 1000  |
+------------------------+-------+
2 rows in set (0.01 sec)

		``` 
##Buffer Poll 预热
* 
```
(root@127.0.0.1) [test]>show global variables like '%innodb_buffer%';
+-------------------------------------+----------------+
| Variable_name                       | Value          |
+-------------------------------------+----------------+
| innodb_buffer_pool_dump_at_shutdown | ON      关机dump       |
| innodb_buffer_pool_dump_now         | OFF            |
| innodb_buffer_pool_filename         | ib_buffer_pool dump文件名 |
| innodb_buffer_pool_load_abort       | OFF    abort时dump        |
| innodb_buffer_pool_load_at_startup  | ON         启动载入    |
| innodb_buffer_pool_load_now         | OFF            |
+-------------------------------------+----------------+
10 rows in set (0.00 sec)
```	
```
可以看到dump的是(space, page_num) 先space、page_num排序 然后在载入内存
root@ubuntu:/mdata/mysql_data# cat ib_buffer_pool
22,35
3,2
3,1
3,3
10,2
10,1
10,3
11,2
11,1
11,3
9,2
9,1
9,3
8,2
8,1
8,3
12,2
```
##压缩
##透明页压缩 TPC
`LZ4` 快 压缩比相对于`ZLIB`大
```
create table b (b int primary key) compression='lz4';
alter table b compression='lz4'
```
[官方文档介绍](https://dev.mysql.com/doc/refman/5.7/en/innodb-page-compression.html)
[官方测试博客](http://mysqlserverteam.com/innodb-transparent-page-compression/)
