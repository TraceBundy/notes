##复制集注意点
###Arbiter 节点
Arbiter 节点是专门用来形成多数派的投票节点，它并不复制数据
```
	注意 不要将Arbiter 节点与主节点放在同一台机器上
```

```
For replica sets with an arbiter, replica set protocol version 1 (pv1) increases the likelihood of rollback of w:1 writes compared to replica set protocol version 0  (pv0). See Replica Set Protocol Versions.
```
官方文档有这样一句话，在`W:1`情况下，复制集协议`version1`比`version0`更容易增加回滚的概率 我们同样可以在官方文档中找到答案


```
Back to Back Elections
To maximize write availability, pv1 does not consider priority when conducting an election. Instead, after a replica set has a stable primary, pv1 makes a “best-effort” attempt to have the secondary with the highestpriority available call an election. This could lead to back-to-back elections as eligible members with higher priority can call an election. But unlike pv0, which must include a 30 second buffer between back-to-back elections, the use of terms in pv1 allows for faster occurence of back-to-back elections.
Both the increased frequency and the lack of a time buffer between back-to-back elections with pv1 increase the likelihood of rollback of w:1 writes. However, you can reduce the number of rollbacks by raising the catchUpTimeoutMillis setting.
During an election, pv0 allows nodes to veto based on priority values. As such, after a replica set has a stable primary, pv0 would lead to less back-to-back elections than pv1. Because pv0 relies on clock synchronization to detect multiple primaries, pv0 includes a 30 seconds buffer between back-to-back elections as a precaution against poor clock synchronization.

```
就是说 在背靠背选举中，`PV1` 中在已经有主节点的情况下，高优先级的从节点可以马上发起选举，`PV0`要等30秒。比如：当在一个稳定的集群中，有持续写入，`PV1`中高优先级的可以马上发起选举，把当前主节点干掉。那么上一个主节点没有复制到其它节点的数据就会被回滚掉， 而`PV0`有30秒的Buffer,所以会减少概率。 那么增加Arbiter 节点，Arbiter节点本身不写入数据，所以在`PV1`概率会加大

```
考虑在只有3节点的复制集，其中有一个Arbiter节点，在主节点挂了，从节点被选为主节点。由于`W:1`有可能主节点上的数据没有被复制到从节点。Arbiter节点不写入数据，所以从节点被选为主节点时，可能会丢失最近的主节点数据。
```

[官方链接 replica-set-protocol-versions](https://docs.mongodb.com/manual/reference/replica-set-protocol-versions/)


