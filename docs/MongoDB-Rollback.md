##MongoDB故障恢复和选举
###选举状态
* Vote 为1的节点均可以选举
* 可进行选举的节点
	* Primary 、secondary
	* Recovering 、 arbiter 、 rollback 可以参与选举，但不能成为候选人(成为主节点)
###影响因素
* 复制选举协议
	* Pv1提高了故障恢复协议， 缩短了多primary的发现时间
* 心跳互探
	默认2秒一次心跳、10秒心跳超时，若超时，则判断该节点不可访问
	```
		"settings" : {
		"chainingAllowed" : true,
		"heartbeatIntervalMillis" : 2000, 心跳
		"heartbeatTimeoutSecs" : 10,心跳超时
		"electionTimeoutMillis" : 10000,
		"catchUpTimeoutMillis" : 2000, 从其它节点拉数据的超时时间
		"getLastErrorModes" : {

		}
	```
* 成员的priority设置
	* 影响选举时间和结果
	* 高priority会更快发起选举、更容易赢得选举
	* 低priority 能获胜，但高priority会继续发起选举直到获胜

	```
		{
			"_id" : 3,
			"host" : "127.0.0.1:47020",
			"arbiterOnly" : false,
			"buildIndexes" : true,
			"hidden" : false,
			"priority" : 1,
			"tags" : {

			},
			"slaveDelay" : NumberLong(0),
			"votes" : 1
		}
	```
	* 跨数据中心
		* 数据中心失联，另一个数据中心的节点会重新选举
		* 确保选出的新primary跟应用在同一个数据中心
		
		```
			最热门的莫过于`两地三中心` ,就是同城有两个数据中心，异地有一个数据中心来进行灾备。例如: 北京有两个数据中心， 每个数据中心有两个副本，异地杭州有一个副本。这时候我们就需要用到priority来进行选举的优化。使同一个数据中心的具有相同有priority， 可设BJ.DC1(priority=2), BJ.DC2(priority=1), HZ.DC3(priority=0)这样当主节点挂掉，新主会在同一数据中心、或者同城数据中心来选出。
		```
	* 网络分区时
		* primary 只能连接上少数节点时，会step down 为secondary
		* secondary 发现无primary 时， 若能连上大多数，会触发选举
		```
			因为主节点不能连上多数节点，则写入的数据不能写入到大多数节点
		```
###数据回滚
* 新primary选出后，旧primary可能需要数据回滚
	```
		旧primary有些数据未能写到majority节点,需要回滚，使得各节点数据一致
	```
* 引起回滚原因
	* 网络分区
		```
		新primary已经产生，旧primary并未感知到，仍然接收客户端请求。这个时间很短暂。
		```
	* primary 压力过大导致secondary无法跟上
* 若primary宕机时， 数据已复制到一个secondary节点，且该节点能访问majority节点，则该数据无需回滚
* 回滚数据会保存在rollback文件夹
	* <database>.<collection>.<timestamp>.bson形式存放
	* 回滚限制300MB 大于300MB则需要人工介入

	
* 如何避免回滚
	* W ： majority
	* writeConcernMajorityJournalDefault:true



