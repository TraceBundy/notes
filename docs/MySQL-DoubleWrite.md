
#DubleWrite
```
在数据中我们的页可能是16K的，但一次写入有可能为4K、6K、8K、12K的情况，这时候是不能用Redo Log来恢复的。为此MySQL引入DoubleWrite用来保证数据写入的可靠性。
```
```
每次更新的时候并不直接写页文件，先写入到doublewrite的表空间里，再写入到数据文件。每次写入doublewrite为128页(doublewrite在共享表空间为2个文件，每个1MB)。
```

##性能开销
```
doublewrite是顺序写入，性能开销取决于写入量，性能开销通常5%~25%，slave服务器可以考虑关闭
```

##关闭参数
```
(root@127.0.0.1) [information_schema]>show global variables like 'innodb_doublewrite%';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| innodb_doublewrite | ON    |
+--------------------+-------+
1 row in set (0.00 sec)

```
#Insert/Change Buffer
* 提升二级非唯一索引的增删改性能

##原理
1、先判断插入的非聚集索引页是否在缓冲池中，若在，则直接插入
2、若不在，则先放入到一个Insert Buffer对象中
	* insert buffer 也是一颗B+树
	* 每次最多缓存2K的记录
3、当读取辅助索引页到缓存池，将Insert Buffer中该页的记录合并到辅助索引页
```
------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1(使用的page), free list len 3092(空闲page), seg size 3094(=size+free list len + 1, 1018710 merges
merged operations:
 insert 25797737, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
Hash table size 34673, node heap has 1 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
```
#潜在问题

* 最大可以使用1/2缓冲池内存

	```
	(root@127.0.0.1) [information_schema]>show global variables like 'innodb_change_buffer_max%';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| innodb_change_buffer_max_size | 25    |
+-------------------------------+-------+
1 row in set (0.01 sec)
	```

* shutdown 不进行insert buffer记录的合并
* insert buffer 开始进行合并时插入性能开始下降

#自适应哈希索引
根据B+树的访问模式构建哈希索引，仅对热点页中的记录创建哈希索引。存放也内存，仅支持点查询
```
(root@127.0.0.1) [information_schema]>show global variables like 'innodb_%hash%';
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| innodb_adaptive_hash_index       | ON    |
| innodb_adaptive_hash_index_parts | 8     |
+----------------------------------+-------+
2 rows in set (0.00 sec)
```
##判断条件
* 索引是否被访问了17次
* 索引中某个页已经被访问了至少100次
* 对索引中的页访问的模式都是相同的
* idx_a_b(a,b) WHERE a=xxx 或 WHERE a = xxx and b = xxx

#FLUSH NEIGHBOR PAGE
试图刷新页所在区中的所有脏页，对传统机械磁盘有效，SSD关闭。
```
(root@127.0.0.1) [information_schema]>show global variables like 'innodb_flush_neighbors%';
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| innodb_flush_neighbors | 1     |
+------------------------+-------+
1 row in set (0.00 sec)
```


