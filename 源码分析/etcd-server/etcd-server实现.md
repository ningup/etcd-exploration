# 1. raftNode  底层raft和上层模块的桥梁
```go
type raftNode struct {
	lg *zap.Logger

    tickMu *sync.Mutex
    // 内嵌了 raftNodeConfig, 后面说明
	raftNodeConfig

    // a chan to send/receive snapshot
    // raftNode 收到 MsgSnap 消息后，会把 Message 发送到该通道，等待上层模块发送
	msgSnapC chan raftpb.Message

    // a chan to send out apply
    // raftNode 会把待应用的 Entry 和 快照数据封装成 apply实例，等待上层模块处理，  apply 后边会说明
	applyc chan apply

    // a chan to send out readState
    // 只读请求相关的通道，也是发给上层模块
	readStateC chan raft.ReadState

    // utility
    // 逻辑时钟定时器
	ticker *time.Ticker
    // contention detectors for raft heartbeat message
    // 检测同一节点的心跳消息是否超时
	td *contention.TimeoutDetector

	stopped chan struct{}
	done    chan struct{}
}

type raftNodeConfig struct {
	lg *zap.Logger

    // to check if msg receiver is removed from cluster
    // 检测指定节点是否移除集群
	isIDRemoved func(id uint64) bool
    raft.Node  // 内嵌了 raft.Node
    // 之前已经讲过了，和 raftLog.storage 指向同一实例，保存待持久化的 Entry 记录和快照数据
    raftStorage *raft.MemoryStorage
    // etcdserver 的 storage，不是raft里边storage 要注意，后面说明
    storage     Storage
    // 逻辑时钟刻度，后面会说明
	heartbeat   time.Duration // for logging
    // 前文描述的用于通过网络发送消息到集群其他节点的
	transport rafthttp.Transporter
}

type Storage interface {
    // 持久化 entry 和hardstate，可能会阻塞，是通过wal模块实现的
	Save(st raftpb.HardState, ents []raftpb.Entry) error
    // SaveSnap function saves snapshot to the underlying stable storage.
    // 持久化快照数据，可能阻塞，有前文的 Snapshotter 将快照保存到快照文件
	SaveSnap(snap raftpb.Snapshot) error
	// Close closes the Storage and performs finalization.
	Close() error
	// Release releases the locked wal files older than the provided snapshot.
	Release(snap raftpb.Snapshot) error
	// Sync WAL
	Sync() error
}
```

## 初始化 raftNode
```go
func newRaftNode(cfg raftNodeConfig) *raftNode {
	r := &raftNode{
		lg:             cfg.lg,
		tickMu:         new(sync.Mutex),
		raftNodeConfig: cfg,
		// set up contention detectors for raft heartbeat message.
		// expect to send a heartbeat within 2 heartbeat intervals.
		td:         contention.NewTimeoutDetector(2 * cfg.heartbeat),
        readStateC: make(chan raft.ReadState, 1),
        // 缓冲区大小是 axInFlightMsgSnap = 16
                     // max number of in-flight snapshot messages etcdserver allows to have
	                 // This number is more than enough for most clusters with 5 machines.
		msgSnapC:   make(chan raftpb.Message, maxInFlightMsgSnap),
		applyc:     make(chan apply),
		stopped:    make(chan struct{}),
		done:       make(chan struct{}),
    }
    // 根据 raftNodeConfig 的 heartbeat 创建逻辑时钟
	if r.heartbeat == 0 {
		r.ticker = &time.Ticker{}
	} else {
        // 刻度时 heartbeat
		r.ticker = time.NewTicker(r.heartbeat)
	}
	return r
}
```

## raftNode.start() 启动相关服务
创建了一个后台协程，用来与底层 raft模块进行交互
```go
func (r *raftNode) start(rh *raftReadyHandler) {
	internalTimeout := time.Second

	go func() {
        defer r.onStop()
        // 刚启动不是leader， 表示为 follower
		islead := false

		for {
			select {
            // 逻辑时钟触发
            case <-r.ticker.C:
                // 调用 raft.Node.Tick()
                r.tick()
            // 处理底层 raft 传过来的 Ready 对象，并进行处理, 具体实现后边说明，这里先跳过
            case rd := <-r.Ready():
                ...
                // 处理完调用 raft.node.Advance() 通知底层 raft 已经处理完该 Ready 实例
				r.Advance()
			case <-r.stopped:
				return
			}
		}
	}()
}
```
下面说明 具体是如果处理 Ready 对象的，代码片段如下:
!![](../img/33.png)
```go
            case rd := <-r.Ready():
				if rd.SoftState != nil {
                    // 检查 leader 是否发生变化，更新 raftNode.lead 字段
					newLeader := rd.SoftState.Lead != raft.None && rh.getLead() != rd.SoftState.Lead
					if newLeader {
						leaderChanges.Inc()
					}

					if rd.SoftState.Lead == raft.None {
						hasLeader.Set(0)
					} else {
						hasLeader.Set(1)
					}
                    // raftReadyHandler 的 updateLead
                    // rh *raftReadyHandler 是 etcdserver 保存节点状态的结构（比如是否leader，entry提交位置等）
                    // raftNode 可以通过 raftReadyHandler 的函数修改 etcdserver 相关的字段
					rh.updateLead(rd.SoftState.Lead)
					islead = rd.RaftState == raft.StateLeader
					if islead {
						isLeader.Set(1)
					} else {
						isLeader.Set(0)
                    }
                    // 更新leadership 的具体操作，后边说明
                    rh.updateLeadership(newLeader)
                    // 重置探测器中的全部记录
					r.td.Reset()
				}
                
				if len(rd.ReadStates) != 0 {
					select {
                        // 将 ReadStates的最后一项写入 readStateC 通道
					case r.readStateC <- rd.ReadStates[len(rd.ReadStates)-1]:
                    case <-time.After(internalTimeout):
                        // 如果上层应用一直没读通道数据，等待超时后会放弃本次写入，打印报警日志
                        ...
					case <-r.stopped:
						return
					}
				}
   
                // 创建 notifyc 通道
                notifyc := make(chan struct{}, 1)
                // 创建 apply 实例
				ap := apply{
					entries:  rd.CommittedEntries,  // 已经提交待应用的 entry
					snapshot: rd.Snapshot,          // 待持久化的快照数据
					notifyc:  notifyc,              // 
				}

                // 更新 etcd server记录的已经提交的位置
				updateCommittedIndex(&ap, rh)

				select {
                    // 向 ap 数据写入 通道，待上层处理
				case r.applyc <- ap:
				case <-r.stopped:
					return
				}
                // 如果时 leader，对待发送的消息进行过滤，然后调用 transport 进行发送
				if islead {
					r.transport.Send(r.processMessages(rd.Messages))
				}

				if !raft.IsEmptySnap(rd.Snapshot) {
                    // 通过 etcdserver 的 storage 将 ready 中的快照数据保存到磁盘, 个人理解是发送的快照数据
					if err := r.storage.SaveSnap(rd.Snapshot); err != nil {
					}
				}

                // 通过 etcdserver 的 storage 持久化 entry 和 hardstate， 具体使用是wal
				if err := r.storage.Save(rd.HardState, rd.Entries); err != nil {
					}
				}
				if !raft.IsEmptyHardState(rd.HardState) {
					proposalsCommitted.Set(float64(rd.HardState.Commit))
				}
				// gofail: var raftAfterSave struct{}

				if !raft.IsEmptySnap(rd.Snapshot) {
                    // 强行同步 wal 等数据到磁盘
					if err := r.storage.Sync(); err != nil {
					}

                    // etcdserver now claim the snapshot has been persisted onto the disk
                    // 通知 上层应用协程，快照数据已经生成
					notifyc <- struct{}{}

                    // 保存快照信息到 MemoryStroage
                    r.raftStorage.ApplySnapshot(rd.Snapshot)


					if err := r.storage.Release(rd.Snapshot); err != nil {
					}
				}
                // 保存待持久化 entry 信息到 MemoryStroage
				r.raftStorage.Append(rd.Entries)

				if !islead {
                    // 不是 leader 的过滤 msg 消息
					msgs := r.processMessages(rd.Messages)

                    // 检测是否生成快照
					notifyc <- struct{}{}

                    // 发送消息
					r.transport.Send(msgs)
				} else {
					// leader already processed 'MsgSnap' and signaled
					notifyc <- struct{}{}
				}

				r.Advance()
```

## etcdserver.storage
```go
type Storage interface {
	// Save function saves ents and state to the underlying stable storage.
	// Save MUST block until st and ents are on stable storage.
	Save(st raftpb.HardState, ents []raftpb.Entry) error
	// SaveSnap function saves snapshot to the underlying stable storage.
	SaveSnap(snap raftpb.Snapshot) error
	// Close closes the Storage and performs finalization.
	Close() error
	// Release releases the locked wal files older than the provided snapshot.
	Release(snap raftpb.Snapshot) error
	// Sync WAL
	Sync() error
}

// 内嵌了前文的 wal 和 snapshotter
type storage struct {
	*wal.WAL
	*snap.Snapshotter
}
```
重写了 savesnap()
```go
func (st *storage) SaveSnap(snap raftpb.Snapshot) error {
    // 创建snapshot 实例
	walsnap := walpb.Snapshot{
		Index: snap.Metadata.Index,
		Term:  snap.Metadata.Term,
    }
    // 将快照写入磁盘
	err := st.Snapshotter.SaveSnap(snap)
	if err != nil {
		return err
	}
	// gofail: var raftBeforeWALSaveSnaphot struct{}

    // 将 walpbsnapshot 写入 wal
	return st.WAL.SaveSnapshot(walsnap)
}
```

# 2. raftCluster 记录集群节点的状态
```go
// RaftCluster is a list of Members that belong to the same raft cluster
type RaftCluster struct {
	lg *zap.Logger

	localID types.ID
	cid     types.ID
	token   string

    // v2 版本的存储
	v2store v2store.Store
	// v3 版本的存储
	be      backend.Backend

	sync.Mutex // guards the fields below
	version    *semver.Version
	// 每个节点都会有一个id 和对应的一个 Member 实例
	members    map[types.ID]*Member
	// removed contains the ids of removed members in the cluster.
	// removed id cannot be reused.
	removed map[types.ID]bool
}
```
已 AddMember 为例
```go
func (c *RaftCluster) AddMember(m *Member) {
	c.Lock()
	defer c.Unlock()
	// 根据后端存储的版本来持久化 Member 信息
	if c.v2store != nil {
		mustSaveMemberToStore(c.v2store, m)
	}
	if c.be != nil {
		mustSaveMemberToBackend(c.be, m)
	}

    // 补全 members 成员信息
	c.members[m.ID] = m

    // 打印相关信息
	if c.lg != nil {
		c.lg.Info(
			"added member",
			zap.String("cluster-id", c.cid.String()),
			zap.String("local-member-id", c.localID.String()),
			zap.String("added-peer-id", m.ID.String()),
			zap.Strings("added-peer-peer-urls", m.PeerURLs),
		)
	} else {
		plog.Infof("added member %s %v to cluster %s", m.ID, m.PeerURLs, c.cid)
	}
}
```