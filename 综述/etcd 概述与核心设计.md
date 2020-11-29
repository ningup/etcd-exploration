[TOC]

# 1 etcd 综述
## 1.1 etcd 是什么
官网定义： 
1. Highly-avaliable key value store for shared configuration and service discovery
2. A distributed, reliable key-value store for the most critical data of a distributed system
* [性能测试](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/performance.md#benchmarks)

## 1.2 Adopters
![](img/1.png)

## 1.3 etcd VS zookeeper
主要解决的问题：
* 元数据存储
* 分布式协同

etcd 的设计简洁，运维简单，zk 设计的比较复杂（没研究过），**所以只列了 etcd 的优点**：
* 支持线性读（支持读到最新数据）
* 高负载下的读写性能稳定
* 流式 Watcher，历史事件不会丢失，复用网络连接；zookeeper 是单次 watch，服务端接受到 watch 请求之前的事件丢失，每个 watch 请求使用新的连接
* 客户端协议：
    * zk 基于自定义的 Jute RPC， 限制了其他语言的支持
	* etcd 基于 gRPC，其他语言开箱即用
* 分布式协同原语维护方（分布式锁，选举，租约等）
    * etcd: 官方维护
	* zk: 通常使用 Curator。Curator是 Netflix 公司开源的一个Zookeeper客户端，Curator框架在zookeeper 原生 API 接口上进行了包装，解决了很多 ZooKeeper 客户端非常底层的细节开发

## 1.4 版本控制 (MVCC)
* 写操作都会创建新的版本
* 读操作会从有限的多个版本中挑选一个最合适的（要么是最新版本，要么是指定版本)
* 无需关心读写之间的冲突

### 1.4.1 版本描述
从 etcd 中描述 key value 可以看到：
```go
type KeyValue struct {
	// key is the key in bytes. An empty key is not allowed.
	Key []byte `protobuf:"bytes,1,opt,name=key,proto3" json:"key,omitempty"`
	// 该 key 最后一次时的 revision, （分布式锁就用了它）
	CreateRevision int64 `protobuf:"varint,2,opt,name=create_revision,json=createRevision,proto3" json:"create_revision,omitempty"`
	// 该 key 最后一次修改时的 revision
	ModRevision int64 `protobuf:"varint,3,opt,name=mod_revision,json=modRevision,proto3" json:"mod_revision,omitempty"`
	// 1. key 当前的版本
	// 2. 任何对 key 的修改都会增加该值
	// 3. 删除 key 会使该值归零
	Version int64 `protobuf:"varint,4,opt,name=version,proto3" json:"version,omitempty"`
	// value is the value held by the key, in bytes.
	Value []byte `protobuf:"bytes,5,opt,name=value,proto3" json:"value,omitempty"`
	// 赋予的租约id，如果是 0 代表没有租约
	Lease int64 `protobuf:"varint,6,opt,name=lease,proto3" json:"lease,omitempty"`
}
```

### 1.4.2 逻辑时钟 (revision)
* int64 全局递增
* 可以视为全局的逻辑时钟，任何键空间发生变化，revision 都会增加；指定 revision 操作时，理解为时钟回退到那个时刻
* 用事务封装的操作，revision 在后台只会更改一次
* revision值大的键值对一定是在revision值小键值对之后修改的

## 1.5 etcd API
rpc pb 描述
* https://github.com/etcd-io/etcd/blob/release-3.4/etcdserver/etcdserverpb/rpc.proto
* https://godoc.org/github.com/coreos/etcd/clientv3

### 1.5.1 核心 API
#### 1.5.1.1 KV
(range, 线性读/串行读)
```go
service KV {
  // 从键值存储中获取范围内的key.
  rpc Range(RangeRequest) returns (RangeResponse) {}
​
  // 放置给定key到键值存储.
  rpc Put(PutRequest) returns (PutResponse) {}
​
  // 从键值存储中删除给定范围。
  rpc DeleteRange(DeleteRangeRequest) returns (DeleteRangeResponse) {}
​
  // 在单个事务中处理多个请求。
  // 不容许在一个txn中多次修改同一个key.
  rpc Txn(TxnRequest) returns (TxnResponse) {}
​
  // 压缩在etcd键值存储中的事件历史。
  // 键值存储应该定期压缩，否则事件历史会无限制的持续增长.
  rpc Compact(CompactionRequest) returns (CompactionResponse) {}
}
```

#### 1.5.1.2 Watch
(复用连接、watch 范围/历史)
```go
service Watch {
  // Watch watches for events happening or that have happened. Both input and output
  // are streams; the input stream is for creating and canceling watchers and the output
  // stream sends events. One watch RPC can watch on multiple key ranges, streaming events
  // for several watches at once. The entire event history can be watched starting from the
  // last compaction revision.
  // 1. Watch 检测将要发生或者已经发生的事件。
  // 2. 输入流用于创建和取消观察，输出流发送事件。
  // 3. 一个 RPC 可以在多个 key range 上 watch。
  // 4. watch 可以在最后压缩修订版本开始观察。
  rpc Watch(stream WatchRequest) returns (stream WatchResponse) {}}
}
```

#### 1.5.1.3 Lease
(有最小TTL/集群时钟不一致时ttl不准)
```go
service Lease {
  // LeaseGrant 创建一个租约，当服务器在给定 time to live 时间内没有接收到 keepAlive 时租约过期。
  // 如果租约过期则所有附加在租约上的key将过期并被删除。
  // 每个过期的key在事件历史中生成一个删除事件。
  rpc LeaseGrant(LeaseGrantRequest) returns (LeaseGrantResponse) {}

  // LeaseRevoke 撤销一个租约。
  // 所有附加到租约的key将过期并被删除。
  rpc LeaseRevoke(LeaseRevokeRequest) returns (LeaseRevokeResponse) {}

  // 续约
  rpc LeaseKeepAlive(stream LeaseKeepAliveRequest) returns (stream LeaseKeepAliveResponse) {}

  // LeaseTimeToLive 获取租约信息。
  rpc LeaseTimeToLive(LeaseTimeToLiveRequest) returns (LeaseTimeToLiveResponse) {}
}
```

clientV3：
* KeepAlive 自动无限续租
    * grpc stream，创建协程无限续租，间隔500ms
    * 如果 stream 失效， 500ms重试
* KeepAliveOnce 续租一次

#### 1.5.1.4 Cluster
集群管理相关 add/remove/update member/learner


#### 1.5.1.5 Maintenance
维护相关操作(报警/后端碎片整理)

#### 1.5.1.6 Auth
用户鉴权管理相关

### 1.5.2 并发 API
[并发API官方文档](https://github.com/etcd-io/etcd/blob/master/Documentation/dev-guide/api_concurrency_reference_v3.md)
#### 1.5.2.1 Lock 分布式锁
```go
type LockServer interface {
	// Lock acquires a distributed shared lock on a given named lock.
	// On success, it will return a unique key that exists so long as the
	// lock is held by the caller. This key can be used in conjunction with
	// transactions to safely ensure updates to etcd only occur while holding
	// lock ownership. The lock is held until Unlock is called on the key or the
	// lease associate with the owner expires.
	Lock(context.Context, *LockRequest) (*LockResponse, error)
	// Unlock takes a key returned by Lock and releases the hold on lock. The
	// next Lock caller waiting for the lock will then be woken up and given
	// ownership of the lock.
	Unlock(context.Context, *UnlockRequest) (*UnlockResponse, error)
}
```
etcdserver/api/v3lock/lock.go 通过 clientV3(concurrency) 实现

* 在 key prefix 下创建一个key，并不断续租
* key prefix下的所有 key 有不同的revision，revision 最小的那个 key 将获得锁
* revision 不是最小的 key 的持有者将阻塞，直到revision比它小的所有key都被删除时，它才获得锁

核心逻辑：
* lock
```go
func (m *Mutex) Lock(ctx context.Context) error {
	s := m.s
	client := m.s.Client()
    //尝试抢锁客户端要创建的key的名字
    m.myKey = fmt.Sprintf("%s%x", m.pfx, s.Lease())
    // 事务，key不存在则创建，存在则get（重复使用已经持有的锁key）,并且都获取 prefix key的信息
	cmp := v3.Compare(v3.CreateRevision(m.myKey), "=", 0)
	put := v3.OpPut(m.myKey, "", v3.WithLease(s.Lease()))
	get := v3.OpGet(m.myKey)
    getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...)
    // 启动事务 
    resp, err := client.Txn(ctx).If(cmp).Then(put, getOwner).Else(get, getOwner).Commit()
    // 拿到当前的 revision，用于比较自己是否是最小的, 如果是最小的则抢锁成功
	m.myRev = resp.Header.Revision
	// if no key on prefix / the minimum rev is key, already hold the lock
	ownerKey := resp.Responses[1].GetResponseRange().Kvs
	if len(ownerKey) == 0 || ownerKey[0].CreateRevision == m.myRev {
		m.hdr = resp.Header
		return nil
	}
    // 如果没抢到就等待比自己小的key都删除
	hdr, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)
	return werr
}
```
问题： 如果 keyprefix下的 key 被外部删除了，所有都认为自己抢锁成功
* unlock: 把 myKey 删掉就行了


#### 1.5.2.2 Election 选举
```go
type ElectionServer interface {
	// Campaign waits to acquire leadership in an election, returning a LeaderKey
	// representing the leadership if successful. The LeaderKey can then be used
	// to issue new values on the election, transactionally guard API requests on
	// leadership still being held, and resign from the election.
	Campaign(context.Context, *CampaignRequest) (*CampaignResponse, error)
	// Proclaim updates the leader's posted value with a new value.
	Proclaim(context.Context, *ProclaimRequest) (*ProclaimResponse, error)
	// Leader returns the current election proclamation, if any.
	Leader(context.Context, *LeaderRequest) (*LeaderResponse, error)
	// Observe streams election proclamations in-order as made by the election's
	// elected leaders.
	Observe(*LeaderRequest, Election_ObserveServer) error
	// Resign releases election leadership so other campaigners may acquire
	// leadership on the election.
	Resign(context.Context, *ResignRequest) (*ResignResponse, error)
}
```
etcdserver/api/v3election/election.go 通过 clientV3(concurrency) 

选举

1. etcd的选举也是在相应的prefix path下面创建key，该key绑定了lease 并根据 lease id进行命名
2. key创建后就有revision号，这样使得在prefix path下的key也都是按revision有序
3. 每个节点watch比自己createRevision小并且最大的节点，等到所有比自己createRevision小的节点都被删除后，自己才成为leader
```go
func (e *Election) Campaign(ctx context.Context, val string) error {
	s := e.session
	client := e.session.Client()
	// 事务，如果if判断为true，那么put这个key，否则get这个key；最终都能获取到这个key的内容。
	k := fmt.Sprintf("%s%x", e.keyPrefix, s.Lease())
	txn := client.Txn(ctx).If(v3.Compare(v3.CreateRevision(k), "=", 0))
	txn = txn.Then(v3.OpPut(k, val, v3.WithLease(s.Lease())))
	txn = txn.Else(v3.OpGet(k))
	resp, err := txn.Commit()

	e.leaderKey, e.leaderRev, e.leaderSession = k, resp.Header.Revision, s
	// key 已经存在，执行的事务的 else 分支
	if !resp.Succeeded {
		kv := resp.Responses[0].GetResponseRange().Kvs[0]
		e.leaderRev = kv.CreateRevision
		if string(kv.Value) != val {
			// 判定val不相同，在不更换leader的情况下，更新val
			if err = e.Proclaim(ctx, val); err != nil {
				e.Resign(ctx)
				return err
			}
		}
	}

    // 一直阻塞知道当前的revision最小,从而当选 leader
	_, err = waitDeletes(ctx, client, e.keyPrefix, e.leaderRev-1)
	return nil
}
```

重新选举
```go
func (e *Election) Resign(ctx context.Context) (err error) {
	if e.leaderSession == nil {
		return nil
	}
	client := e.session.Client()
	// 如果当前leaderkey的revision没有变化，就把节点删除，
	cmp := v3.Compare(v3.CreateRevision(e.leaderKey), "=", e.leaderRev)
	resp, err := client.Txn(ctx).If(cmp).Then(v3.OpDelete(e.leaderKey)).Commit()
	e.leaderKey = ""
	e.leaderSession = nil
	return err
}
```

## 1.6 etcd 主要应用场景

* 服务发现 (租约，心跳保持 )
* 消息发布与订阅 (Watch)
* 负载均衡 (利用服务发现维护可用服务列表，请求过来后轮询转发)
* 分布式通知与协调 (Watch)
* 分布式锁 (最小的 create_revision)
* 选举（分布式锁）

## 1.7 Demo
https://etcd.io/docs/v3.4.0/demo/
* get/put
* compact
* txn
* watch
* lease
* status
```shell
[DEV (v.v) sa_cluster@test1 ~]$ /sensorsdata/main/program/armada/etcd/bin/etcdctl endpoint status -w="table"  --endpoints=test1.sa:2379
+---------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|   ENDPOINT    |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| test1.sa:2379 | 685785ec03d61bfd |  3.4.13 |   20 kB |      true |      false |         2 |          8 |                  8 |        |
+---------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

# 2 总体架构
# 3 内部机制解析
## 3.1 共识层（raft）
## 3.2 网络层 (raft-http)
## 3.3 wal 
## 3.4 snap
## 3.5 存储层 (MVCC)
## 3.6 持久层（boltdb）
## 3.7 租约（leasor）
## 3.8 监听（watcher）
## 3.9 服务端（etcd server）
## 3.10 客户端 （etcd client）


