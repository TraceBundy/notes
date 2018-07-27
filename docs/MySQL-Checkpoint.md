#Checkpoint
* 缩短数据库的恢复时间
* 缓冲池不够用时，将脏页刷新到磁盘
* 重做日志不可用时，刷新脏页
```
LSN有三个位置存放

	* 每个页
	* Redo Log 
	* 每个数据库都有 全局LSN
```
`SHOW ENGINE INNODB STATUS;`
```
---
LOG
---
Log sequence number 9966946860 内存产生的LSN
Log flushed up to   9966946860 刷新日志文件的LSN位置
Pages flushed up to 9966946860 最大的页刷新的LSN位置
Last checkpoint at  9966946851 我当前数据库的LSN位置
0 pending log flushes, 0 pending chkp writes
27905 log i/o's done, 0.00 log i/o's/second
```

######为什么上面4个LSN值不一样 `Last checkpoint` 小于其余三个?
```
Buffer Pool 里的Flush List 是按照LSN来排序的，但同一个页中的LSN只记录的是第一次放进去的LSN位置。那么当刷入到磁盘内的时候，记录的也是第一次的LSN位置。不会记录当前最新的LSN位置。所以会小。
```
##Sharp Checkpoint
将所有的脏页都刷新回磁盘，刷新时系统hang住，在InnoDB关闭时使用 --innodb_fast_shutdown = {1|0}

## Fuzzy Checkpoint
* 将部分脏页刷新回磁盘
* 对系统影响较小
通过 `innodb_io_capacity` 来控制每次刷盘的页数

##Checkpoint的时机
* Master Thread Checkpoint
	* 从FLUSH_LIST中进行刷新
	
* FLUSH_LRU_LIST Checkpoint
	* LRU需要有差不多100个空闲页
	
* Async/Sync Flush Checkpoint
	* 重做日志重用
	
* Dirty Page too much Checkpoint
	* --innodb_max_dirty_pages_pct

BufferPool中LRU List里面既有干净的页也有脏页。当LRU List页满了的时候，前后要淘汰的页是脏页，那么就需要刷新脏页。通过`innodb_lru_scan_depth`来控制这种情况下刷新脏页的数量
```
root@127.0.0.1) [information_schema]>show global variables like 'innodb_lru%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| innodb_lru_scan_depth | 1024  |
+-----------------------+-------+
1 row in set (0.01 sec)
```





