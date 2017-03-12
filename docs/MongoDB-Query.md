##MongoDB查询
#####Find函数
比较操作有`$eq`,`$gt`,`$gte`,`$lt`,`$lte`,`$ne`,`$in`,`$nin`
查询方式`{ <field>: { $?: <value> } }`
`$eq`查询:
  `{ <field>: { $eq: <value> } }`等价`{ field: <value> }`
#####在find函数中，查找数组中的元素用`[]`例:`{field:["A","B"]}`
查找嵌套元素利用`.`来查找例:`{"item.tag":"A"}`
####逻辑操作符:
`$or`,`$and`,`$not`,`$nor`
`{ $?: [ { <expression1> }, { <expression2> }, ... , { <expressionN> } ] }
`
注意在用`$or`查找同一个字段的多个值时,可以使用`$in`来代替
####元素查询:
`$exists`,`$type`
`$exists`判断文档中包含指定字段，其中为true时值为null也会被匹配到
`{ field: { $exists: <boolean> } }`
`$type`返回文档中指定字段属于指定的类型
`{ field: { $type: <BSON type number> | <String alias> } }`
####估值操作符
`$mod`,`$regex`,`$text`,`$where`
`$mod` 取模：`{ field: { $mod: [ divisor, remainder ] } }`
`$regex` : `{ <field>: { $regex: /pattern/, $options: '<options>' } }` 选项：`i`忽略大小写 `m`多个起始点匹配 `x`扩展 `s`允许'.*'来匹配
`$text` `$where`
####地理位置
`geoWithin` 查找在指定范围内的数据
`geoIntersects`查找指定范围内相交的数据
`near` 有近到远返回指定位置相近的数据，可以是GeoJSON也可以是数组坐标
`nearSphere`有近到远返回指定位置相近的数据，可以是GeoJSON也可以是数组坐标。`nearSphere`球面计算方式
####投射
投射操作符`$` `$elemMatch` `$meta` `$slice`
`$`操作符返回匹配数组中的第一个元素
`$elemMatch` 操作符返回匹配数组中的字段的第一个元素
`$meta`操作符返回文本相关性的值
`$slice`返回指定数量的元素
##索引策略
####创建索引需要考虑的因素
```
* 应用当前及预期的查询操作类型
* 应用查询的读写比
* 系统可用内存情况

优化查询语句 尽可能的全覆盖 或覆盖前缀
```
####最佳判断方式
```
* 建立不同索引并模拟测试和分析实际表现
```
####查询中索引作用行为
```
*绝大多数查询仅使用一个索引
*当查询中包括$or时，每个条件可能使用不同索引
*查询不同字段时，可能出现索引交替使用的情况
```
####创建满足查询需求的索引
```
*如果查询都使用相同字段，则建立单字段索引
*如果还走其它字段，则可建立复合索引
	*也可分别建立索引
*尽可能实现覆盖查询，无需回表，提升效率
```


#### 在区分度高的字段上创建索引
```
*这个字段出现重复值的概率比较小。通过查询条件能准确的查询到数据在分片的片键选择上对区分度有较高要求
* 尽可能的保证索引都尽可能的放入内存减少IO 
```
####使用索引进行结果排序
```
	*即带排序的查询是否走索引查询
		*不走索引的内存排序，使用内存量不能超过32MB
	*单字段索引排序，索引匹配即可
		*可升序、降序
	*复合索引要注意排序顺序，如{a:1,b:1}
		*{a:1,b:1} OK
		*{b:1,a:1} NO
		*{a:-1,b:-1}OK
		*{a:1,b:-1} NO
		*{a:-1,b:1} NO
	*使用索引前缀查询，{a:1,b:1,c:1,d:1}
		*db.data.find().sort({a:1})  prefix{a:1}
		*db.data.find().sort({a:-1}) prefix{a:1}
		*db.data.find().sort({a:1,b:1}) prefix{a:1,b:1}
		*db.data.find().sort({a:-1,b:-1}) prefix{a:1,b:1}
		*db.data.find().sort({a:1,b:1,c:1}) prefix{a:1,b:1}
		*db.data.find({a:{$gt:4}).sort({a:1,b:1}) prefix{a:1,b:1}
		*db.data.find({a:5}).sort({b:1,c:1}) prefix {a:1,b:1,c:1}
		*db.data.find({b:3,a:4}).sort({c:1}) prefix{a:1,b:1,c:1}
		*db.data.find({a:5,b:{$lt:3}}).sort({b:1}) prefix{a:1,b:1}
		*db.data.find({a:{$gt:2}}).sort({c:1}) 查询条件a能走索引，排序则不能
		*db.data.find({c:5}).sort({c:1}) 查询条件和排序都不能排序
```




