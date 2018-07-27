#页
```
页是最小的I/O操作单位
```
普通用户表
```
* 默认每个页16K
* --innodb_page_size(from 5.6)
```
压缩表
```
* 基于页的压缩
* 每个表的页大小可以不同
```

```
(root@127.0.0.1) [information_schema]>desc information_schema.INNODB_CMPMEM;
+----------------------+------------+------+-----+---------+-------+
| Field                | Type       | Null | Key | Default | Extra |
+----------------------+------------+------+-----+---------+-------+
| page_size            | int(5)     | NO   |     | 0       |       |
| buffer_pool_instance | int(11)    | NO   |     | 0       |       |
| pages_used           | int(11)    | NO   |     | 0       |       |
| pages_free           | int(11)    | NO   |     | 0       |       |
| relocation_ops       | bigint(21) | NO   |     | 0       |       |
| relocation_time      | int(11)    | NO   |     | 0       |       |
+----------------------+------------+------+-----+---------+-------+
6 rows in set (0.00 sec)
```

##建立压缩表
```
ALTER TABLE XXXX ENGINE=InnoDB ROW_FORMAT=compressed, KEY_BLOCK_SIZE=4
```
原来KEY_BLOCK_SIZE为16K, 现在压缩设置为16K有没有意义？
```
有意义，页内的数据都是压缩的，相同的页大小里能存放更多数据
```
##表空间--记录
* 用户记录保存在数据页中
* 记录格式由ROW_FORMAT选项决定
	* REDUDENT: 兼容老版本InnoDB
	* COMPACT: 默认格式
	* COMPRESSED : 支持压缩
	* DYNAMIC:大对象记录优化
	
	COMPACT对大对象的存储
	
	id| message(768)| 指针(20) 指向行溢出页
	-----|----|----
	DYNAMIC大对象存储
	
	id|指针(20) 指向行溢出页
	------|------
	
	单行记录大小大于`page/2`就会启用行溢出页来进行存储
	
	
	例子:
	一条记录四个字段长度如下
	
	1K| 2K | 3K | 4K
	----|-------|------|----
	那么会将4K记录会记录到行溢出页中， 如果这条记录还是大于`page/2`则递归之前的字段放入溢出页
	
##页格式
`row offset array`InnoDB为索引组织表，所以在页内为有序的。用来快速定位到指定记录。二分查找

##隐藏列
* rowid 6字节
* trxid 6字节
* rollptr 回滚指针 7字节
