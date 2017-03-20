#Oplog详解
##oplog简介
oplog是记录MongoDB修改操作的日志，它位于local数据下的oplog.rs表中。oplog有如下几个特性
```
* 专用的固定集合
* 是secondary节点复制primary数据的媒介
* 复制集成员间可相互访问oplog
* 日志操作具有幂等性
* 集合默认大小为
	存储空间的5%
	最大50GB
	至少应该保持24小时内的数据更改
	节点初始化后不能在线修改
```
oplog的大小有很多考量，设置太小，则有可能日志为保存24小时就被删除，日志太大，比较占用空间。oplog有点像MySQL binlog的Row格式，具有幂等性。在MySQL binlog的日志中，会将一条SQL中的多条修改操作，分别记录。想象一下，如果频繁的进行更新，这会极大的增加日志的大小。
在这几种业务场景中就要增大oplog
```
* 一次更新多个文档时
* 更新操作占较大比例时
* 删除和插入操作比例相似时
```
##oplog日志格式分析
我将会对常见的几种oplog日志格式进行分析。
###oplog字段
```
"ts":时间戳
"t":term 任期
"h":唯一key,标记这条日志
"v":版本号
"op":操作类型
"ns":namespace
"o":具体的操作
```

####插入日志
```
{ "ts" : Timestamp(1490015910, 2), "t" : NumberLong(1), "h" : NumberLong("9036545109967206005"), "v" : 2, "op" : "i", "ns" : "test.first", "o" : { "_id" : ObjectId("58cfd6a65f4b28086ebbeb9b"), "a" : 1, "b" : 2, "c" : 3 } }

插入操作时候，”op"为 i 表明是insert操作 “o"表示了具体insert的数据
```
```
{ "ts" : Timestamp(1490016548, 1), "t" : NumberLong(1), "h" : NumberLong("-5944877628764308640"), "v" : 2, "op" : "u", "ns" : "test.first", "o2" : { "_id" : ObjectId("58cfd6a65f4b28086ebbeb9b") }, "o" : { "$set" : { "d" : 4 } } }


```
`op:u`表明是更新操作， `o2`表明的是更新数据的查询条件

```
{ "ts" : Timestamp(1490016548, 1), "t" : NumberLong(1), "h" : NumberLong("-5944877628764308640"), "v" : 2, "op" : "u", "ns" : "test.first", "o2" : { "_id" : ObjectId("58cfd6a65f4b28086ebbeb9b") }, "o" : { "$set" : { "d" : 4 } } }

```
这是更新条件是$set的情况下
```
 { "ts" : Timestamp(1490016786, 1), "t" : NumberLong(1), "h" : NumberLong("-9134992351700501357"), "v" : 2, "op" : "d", "ns" : "test.first", "o" : { "_id" : ObjectId("58cfd6a65f4b28086ebbeb9b") } }
```
`op:d`表明是delete操作 `o`表明要删除的数据唯一`_id`


```
{ "ts" : Timestamp(1490018643, 1), "t" : NumberLong(1), "h" : NumberLong("4329494339082324243"), "v" : 2, "op" : "c", "ns" : "test.$cmd", "o" : { "create" : "second", "idIndex" : { "v" : 2, "key" : { "_id" : 1 }, "name" : "_id_", "ns" : "test.second" } } }

```
`op:c`表明执行的是`$cmd`操作，上面是`create`操作

```
{ "ts" : Timestamp(1490018767, 1), "t" : NumberLong(1), "h" : NumberLong("-8887964685293209245"), "v" : 2, "op" : "c", "ns" : "test.$cmd", "o" : { "drop" : "second" } }
```
上面是`drop`操作

```
{ "ts" : Timestamp(1490025693, 1), "t" : NumberLong(1), "h" : NumberLong("-922188827512825704"), "v" : 2, "op" : "n", "ns" : "", "o" : { "msg" : "Reconfig set", "version" : 4 } }
```
```
conf = rs.conf()
conf.member[2].priority = 0;
conf.member[2].hidden = false;
rs.reconfig(conf);
```
我将某个从节点的hidden属性设置为true,更新配置文件后



```
{ "ts" : Timestamp(1490026343, 1), "t" : NumberLong(1), "h" : NumberLong("-4483740141190027818"), "v" : 2, "op" : "n", "ns" : "", "o" : { "msg" : "Reconfig set", "version" : 381389 } }
```
```
{ "ts" : Timestamp(1490026379, 1), "t" : NumberLong(1), "h" : NumberLong("-3920023594043113832"), "v" : 2, "op" : "n", "ns" : "", "o" : { "msg" : "Reconfig set", "version" : 381390 } }
```
这是连续两次reconf操作 两个版本号是连续的

```
{ "ts" : Timestamp(1490025978, 1), "t" : NumberLong(1), "h" : NumberLong("1786566620104483541"), "v" : 2, "op" : "n", "ns" : "", "o" : { "msg" : "Reconfig set", "version" : 86841 } }
```
我们将某一节点remove, 注意`o.version`版本号的变化


