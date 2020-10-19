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

# 3. EtcdServer
```go
// EtcdServer is the production implementation of the Server interface
// 核心字段
type EtcdServer struct {
	// inflightSnapshots holds count the number of snapshots currently inflight.
	// 发送出去但是未响应的快照数据
	inflightSnapshots int64  // must use atomic operations to access; keep 64-bit aligned.
	// 节点应用的最大索引
	appliedIndex      uint64 // must use atomic operations to access; keep 64-bit aligned.
	// 节点提交的最大索引
	committedIndex    uint64 // must use atomic operations to access; keep 64-bit aligned.
	term              uint64 // must use atomic operations to access; keep 64-bit aligned.
	lead              uint64 // must use atomic operations to access; keep 64-bit aligned.

	// consistIndex used to hold the offset of current executing entry
	// It is initialized to 0 before executing any entry.
	consistIndex consistentIndex // must use atomic operations to access; keep 64-bit aligned.
	// etcdserver.raftNode
	r            raftNode        // uses 64-bit atomics; keep 64-bit aligned.

    // 节点自身的信息推送给其他节点后，会关闭通道，相当于是对外服务的一个信号
	readych chan struct{}
	Cfg     ServerConfig

	lgMu *sync.RWMutex
	lg   *zap.Logger

    // 负责协调多个协程之间的执行
	w wait.Wait

	readMu sync.RWMutex
	// read routine notifies etcd server that it waits for reading by sending an empty struct to
	// readwaitC
	// 用来协调线性读相关的协程
	readwaitc chan struct{}
	// readNotifier is used to notify the read routine that it can process the request
	// when there is no error
	readNotifier *notifier

	// stop signals the run goroutine should shutdown.
	stop chan struct{}
	// stopping is closed by run goroutine on shutdown.
	stopping chan struct{}
	// done is closed when all goroutines from start() complete.
	done chan struct{}
	// leaderChanged is used to notify the linearizable read loop to drop the old read requests.
	leaderChanged   chan struct{}
	leaderChangedMu sync.RWMutex

	errorc     chan error
	id         types.ID
	attributes membership.Attributes

    // 记录集群全部节点信息
	cluster *membership.RaftCluster

    // v2 版本存储
	v2store     v2store.Store
	// 用来读写快照文件
	snapshotter *snap.Snapshotter

    // 应用v2版本的 entry
	applyV2 ApplierV2

	// applyV3 is the applier with auth and quotas
    // 应用v3版本的 entry
	applyV3 applierV3
	// applyV3Base is the core applier without auth or quotas
	applyV3Base applierV3
	applyWait   wait.WaitTime

    // 支持watch机制的 v3版本存储
	kv         mvcc.ConsistentWatchableKV
	// 租约管理实例
	lessor     lease.Lessor
	bemu       sync.Mutex
	// v3版本后端存储
	be         backend.Backend
	// backend 上边封装的一层存储，用于权限控制
	authStore  auth.AuthStore
	// backend 上边封装的一层存储，用于记录报警
	alarmStore *v3alarm.AlarmStore

	stats  *stats.ServerStats
	lstats *stats.LeaderStats

    // 控制leader定期发送 sync消息的频率
	SyncTicker *time.Ticker
	// compactor is used to auto-compact the KV.
	// 控制 leader 定期要锁存储的频率
	compactor v3compactor.Compactor

	// peerRt used to send requests (version, lease) to peers.
	peerRt   http.RoundTripper
	// 用于生成请求的唯一表示
	reqIDGen *idutil.Generator

	// forceVersionC is used to force the version monitor loop
	// to detect the cluster version immediately.
	forceVersionC chan struct{}

	// wgMu blocks concurrent waitgroup mutation while server stopping
	wgMu sync.RWMutex
	// wg is used to wait for the go routines that depends on the server state
	// to exit when stopping the server.
	// 控制后台协程退出的
	wg sync.WaitGroup

	// ctx is used for etcd-initiated requests that may need to be canceled
	// on etcd server shutdown.
	ctx    context.Context
	cancel context.CancelFunc

	leadTimeMu      sync.RWMutex
	leadElectedTime time.Time

	*AccessController
}
```

## 初始化 server
```go
// NewServer creates a new EtcdServer from the supplied configuration. The
// configuration is considered static for the lifetime of the EtcdServer.
func NewServer(cfg ServerConfig) (srv *EtcdServer, err error) {
	// 初始化V2存储
	st := v2store.New(StoreClusterPrefix, StoreKeysPrefix)

	var (
		// 管理 wal 日志
		w  *wal.WAL
		// raft.Node
		n  raft.Node
		// memortStorage
		s  *raft.MemoryStorage
		// 节点id
		id types.ID
		//集群所有成员信息
		cl *membership.RaftCluster
	)

    // 创建数据目录
	if terr := fileutil.TouchDirAll(cfg.DataDir); terr != nil {
		return nil, fmt.Errorf("cannot access data directory: %v", terr)
	}

    // 是否有 wal 目录
	haveWAL := wal.Exist(cfg.WALDir())

    // 确定快照目录是否存在
	if err = fileutil.TouchDirAll(cfg.SnapDir()); err != nil {
		...
	}
	//快照管理实例，读写快照
	ss := snap.New(cfg.Logger, cfg.SnapDir())

    // boltdb 数据库文件 filepath.Join(c.SnapDir(), "db")
	bepath := cfg.backendPath()
	// 检测数据库文件是否存在
	beExist := fileutil.Exist(bepath)
	// 创建后端存储实例
	be := openBackend(cfg)

    // 主要负责网络请求等功能
	prt, err := rafthttp.NewRoundTripper(cfg.PeerTLSInfo, cfg.peerDialTimeout())
	var (
		remotes  []*membership.Member
		snapshot *raftpb.Snapshot
	)
    // 根据是否存在 wal 目录， 分三种场景
	switch {
		// 没有wal目录且当前节点是正在加入一个运行的集群
	case !haveWAL && !cfg.NewCluster:
	    // 检查配置是否合法
		if err = cfg.VerifyJoinExisting(); err != nil {
			return nil, err
		}
		// 根据配置信息，创建 raftCluster 实例和其中的Member实例
		cl, err = membership.NewClusterFromURLsMap(cfg.Logger, cfg.InitialClusterToken, cfg.InitialPeerURLsMap)
		// 从其他节点请求(http,  "root_url/members")获取集群信息并创建相应的 raftCLuster
		existingCluster, gerr := GetClusterFromRemotePeers(cfg.Logger, getRemotePeerURLs(cl, cfg.Name), prt)
        // 保证版本兼容
		if err = membership.ValidateClusterAndAssignIDs(cfg.Logger, cl, existingCluster); err != nil {
			return nil, fmt.Errorf("error validating peerURLs %s: %v", existingCluster, err)
		}
		if !isCompatibleWithCluster(cfg.Logger, cl, cl.MemberByName(cfg.Name).ID, prt) {
			return nil, fmt.Errorf("incompatible with current running cluster")
		}

        // 更新raftCluster集群id
		remotes = existingCluster.Members()
		cl.SetID(types.ID(0), existingCluster.ID())
		// 设置 V2 存储
		cl.SetStore(st)
		// 设置 V3 存储
		cl.SetBackend(be)
		// 启动 raftNode, 后边会说明
		id, n, s, w = startNode(cfg, cl, nil)
		cl.SetID(id, existingCluster.ID())

	// 没有wal目录且是新建集群
	case !haveWAL && cfg.NewCluster:
	    // 检查配置
		if err = cfg.VerifyBootstrap(); err != nil {
			return nil, err
		}
		// 同理创建 raftCluster 实例
		cl, err = membership.NewClusterFromURLsMap(cfg.Logger, cfg.InitialClusterToken, cfg.InitialPeerURLsMap)
		// 获取当前节点的 Member 实例
		m := cl.MemberByName(cfg.Name)
		// 从集群中其他节点获取集群信息，检测是否有重名 
		if isMemberBootstrapped(cfg.Logger, cl, cfg.Name, prt, cfg.bootstrapTimeout()) {
			return nil, fmt.Errorf("member %s has already been bootstrapped", m.ID)
		}
		// 是否使用 discover 模式
		if cfg.ShouldDiscover() {
			...
		}
		cl.SetStore(st)
		cl.SetBackend(be)
		// 启动节点
		id, n, s, w = startNode(cfg, cl, cl.MemberIDs())
		cl.SetID(id, cl.ID())

	// 有 wal 目录
	// 主要是通过快照恢复 V2v3存储，然后恢复 raft.Node 实例
	// V2存储就是加载json，v3存储恢复是用可用的 boltdb数据库文件
	case haveWAL:
	    // 检测目录可写
		if err = fileutil.IsDirWriteable(cfg.MemberDir()); err != nil {
			return nil, fmt.Errorf("cannot write to member directory: %v", err)
		}

		if err = fileutil.IsDirWriteable(cfg.WALDir()); err != nil {
			return nil, fmt.Errorf("cannot write to WAL directory: %v", err)
		}

		// Find a snapshot to start/restart a raft node
		// 查找合法的快照
		walSnaps, serr := wal.ValidSnapshotEntries(cfg.Logger, cfg.WALDir())
		if serr != nil {
			return nil, serr
		}
		// snapshot files can be orphaned if etcd crashes after writing them but before writing the corresponding
		// wal log entries
		// 加载wal快照
		snapshot, err = ss.LoadNewestAvailable(walSnaps)
		if err != nil && err != snap.ErrNoSnapshot {
			return nil, err
		}

        // 如果快照不为空
		if snapshot != nil {
			// 恢复成V2类型存储
			if err = st.Recovery(snapshot.Data); err != nil {
			}
			// 恢复成V23型存储
			if be, err = recoverSnapshotBackend(cfg, be, *snapshot); err != nil {
			}
		}
        // 重启 raftNode
		if !cfg.ForceNewCluster {
			id, cl, n, s, w = restartNode(cfg, snapshot)
		} else {
			id, cl, n, s, w = restartAsStandaloneNode(cfg, snapshot)
		}

		cl.SetStore(st)
		cl.SetBackend(be)
		//  从v2存储回复集群信息会检测 wal日志和快照数据版本兼容性
		cl.Recover(api.UpdateCapability)
		if cl.Version() != nil && !cl.Version().LessThan(semver.Version{Major: 3}) && !beExist {
			os.RemoveAll(bepath)
			return nil, fmt.Errorf("database file (%v) of the backend is missing", bepath)
		}

    // 不支持其他场景
	default:
		return nil, fmt.Errorf("unsupported bootstrap config")
	}

	if terr := fileutil.TouchDirAll(cfg.MemberDir()); terr != nil {
		return nil, fmt.Errorf("cannot access member directory: %v", terr)
	}

	sstats := stats.NewServerStats(cfg.Name, id.String())
	lstats := stats.NewLeaderStats(id.String())

	heartbeat := time.Duration(cfg.TickMs) * time.Millisecond
	//  创建 etcdServer 实例
	srv = &EtcdServer{
		readych:     make(chan struct{}),
		Cfg:         cfg,
		lgMu:        new(sync.RWMutex),
		lg:          cfg.Logger,
		errorc:      make(chan error, 1),
		v2store:     st,
		snapshotter: ss,
		r: *newRaftNode(
			raftNodeConfig{
				lg:          cfg.Logger,
				isIDRemoved: func(id uint64) bool { return cl.IsIDRemoved(types.ID(id)) },
				Node:        n,
				heartbeat:   heartbeat,
				raftStorage: s,
				storage:     NewStorage(w, ss),
			},
		),
		id:               id,
		attributes:       membership.Attributes{Name: cfg.Name, ClientURLs: cfg.ClientURLs.StringSlice()},
		cluster:          cl,
		stats:            sstats,
		lstats:           lstats,
		SyncTicker:       time.NewTicker(500 * time.Millisecond),
		peerRt:           prt,
		reqIDGen:         idutil.NewGenerator(uint16(id), time.Now()),
		forceVersionC:    make(chan struct{}),
		AccessController: &AccessController{CORS: cfg.CORS, HostWhitelist: cfg.HostWhitelist},
	}
	serverID.With(prometheus.Labels{"server_id": id.String()}).Set(1)

	srv.applyV2 = &applierV2store{store: srv.v2store, cluster: srv.cluster}

	srv.be = be
	minTTL := time.Duration((3*cfg.ElectionTicks)/2) * heartbeat

	// always recover lessor before kv. When we recover the mvcc.KV it will reattach keys to its leases.
	// If we recover mvcc.KV first, it will attach the keys to the wrong lessor before it recovers.
	srv.lessor = lease.NewLessor(
		srv.getLogger(),
		srv.be,
		lease.LessorConfig{
			MinLeaseTTL:                int64(math.Ceil(minTTL.Seconds())),
			CheckpointInterval:         cfg.LeaseCheckpointInterval,
			ExpiredLeasesRetryInterval: srv.Cfg.ReqTimeout(),
		})

	tp, err := auth.NewTokenProvider(cfg.Logger, cfg.AuthToken,
		func(index uint64) <-chan struct{} {
			return srv.applyWait.Wait(index)
		},
		time.Duration(cfg.TokenTTL)*time.Second,
	)
	if err != nil {
		if cfg.Logger != nil {
			cfg.Logger.Warn("failed to create token provider", zap.Error(err))
		} else {
			plog.Warningf("failed to create token provider,err is %v", err)
		}
		return nil, err
	}
	srv.authStore = auth.NewAuthStore(srv.getLogger(), srv.be, tp, int(cfg.BcryptCost))

	srv.kv = mvcc.New(srv.getLogger(), srv.be, srv.lessor, srv.authStore, &srv.consistIndex, mvcc.StoreConfig{CompactionBatchLimit: cfg.CompactionBatchLimit})
	if beExist {
		kvindex := srv.kv.ConsistentIndex()
		// TODO: remove kvindex != 0 checking when we do not expect users to upgrade
		// etcd from pre-3.0 release.
		if snapshot != nil && kvindex < snapshot.Metadata.Index {
			if kvindex != 0 {
				return nil, fmt.Errorf("database file (%v index %d) does not match with snapshot (index %d)", bepath, kvindex, snapshot.Metadata.Index)
			}
			if cfg.Logger != nil {
				cfg.Logger.Warn(
					"consistent index was never saved",
					zap.Uint64("snapshot-index", snapshot.Metadata.Index),
				)
			} else {
				plog.Warningf("consistent index never saved (snapshot index=%d)", snapshot.Metadata.Index)
			}
		}
	}
	newSrv := srv // since srv == nil in defer if srv is returned as nil
	defer func() {
		// closing backend without first closing kv can cause
		// resumed compactions to fail with closed tx errors
		if err != nil {
			newSrv.kv.Close()
		}
	}()

	srv.consistIndex.setConsistentIndex(srv.kv.ConsistentIndex())
	if num := cfg.AutoCompactionRetention; num != 0 {
		srv.compactor, err = v3compactor.New(cfg.Logger, cfg.AutoCompactionMode, num, srv.kv, srv)
		if err != nil {
			return nil, err
		}
		srv.compactor.Run()
	}

	srv.applyV3Base = srv.newApplierV3Backend()
	if err = srv.restoreAlarms(); err != nil {
		return nil, err
	}

	if srv.Cfg.EnableLeaseCheckpoint {
		// setting checkpointer enables lease checkpoint feature.
		srv.lessor.SetCheckpointer(func(ctx context.Context, cp *pb.LeaseCheckpointRequest) {
			srv.raftRequestOnce(ctx, pb.InternalRaftRequest{LeaseCheckpoint: cp})
		})
	}

	// TODO: move transport initialization near the definition of remote
	// 启动 rafthttpTransport 实例
	tr := &rafthttp.Transport{
		Logger:      cfg.Logger,
		TLSInfo:     cfg.PeerTLSInfo,
		DialTimeout: cfg.peerDialTimeout(),
		ID:          id,
		URLs:        cfg.PeerURLs,
		ClusterID:   cl.ID(),
		Raft:        srv,
		Snapshotter: ss,
		ServerStats: sstats,
		LeaderStats: lstats,
		ErrorC:      srv.errorc,
	}
	if err = tr.Start(); err != nil {
		return nil, err
	}
	// add all remotes into transport
	for _, m := range remotes {
		if m.ID != id {
			tr.AddRemote(m.ID, m.PeerURLs)
		}
	}
	for _, m := range cl.Members() {
		if m.ID != id {
			tr.AddPeer(m.ID, m.PeerURLs)
		}
	}
	srv.r.transport = tr

	return srv, nil
}
```
startNode()
```go
func startNode(cfg ServerConfig, cl *membership.RaftCluster, ids []types.ID) (id types.ID, n raft.Node, s *raft.MemoryStorage, w *wal.WAL) {
	var err error
	member := cl.MemberByName(cfg.Name)
	// 节点信息序列化
	metadata := pbutil.MustMarshal(
		&pb.Metadata{
			NodeID:    uint64(member.ID),
			ClusterID: uint64(cl.ID()),
		},
	)
	// 创建 wal日志文件，将上边的元数据信息写入第一条
	if w, err = wal.Create(cfg.Logger, cfg.WALDir(), metadata); err != nil {
	}
	if cfg.UnsafeNoFsync {
		w.SetUnsafeNoFsync()
	}
	// 每个节点创建 peer 实例
	peers := make([]raft.Peer, len(ids))
    // 新建 MemortStorage 实例
	s = raft.NewMemoryStorage()
	// 初始化raft 的配置实例
	c := &raft.Config{
		ID:              uint64(id),
		ElectionTick:    cfg.ElectionTicks,
		HeartbeatTick:   1,
		Storage:         s,
		MaxSizePerMsg:   maxSizePerMsg,
		MaxInflightMsgs: maxInflightMsgs,
		CheckQuorum:     true,
		PreVote:         cfg.PreVote,
	}
	// 启动 raft Node
	if len(peers) == 0 {
		n = raft.RestartNode(c)
	} else {
		n = raft.StartNode(c, peers)
	}
	raftStatusMu.Lock()
	raftStatus = n.Status
	raftStatusMu.Unlock()
	return id, n, s, w
```

restartNode()
```go
func restartNode(cfg ServerConfig, snapshot *raftpb.Snapshot) (types.ID, *membership.RaftCluster, raft.Node, *raft.MemoryStorage, *wal.WAL) {
	var walsnap walpb.Snapshot
	if snapshot != nil {
		walsnap.Index, walsnap.Term = snapshot.Metadata.Index, snapshot.Metadata.Term
	}
	// 根据快照的元数据查找合适的 wal 日志并进行 wal日志的回收
	w, id, cid, st, ents := readWAL(cfg.Logger, cfg.WALDir(), walsnap, cfg.UnsafeNoFsync)

	cl := membership.NewCluster(cfg.Logger, "")
	cl.SetID(id, cid)
	// 创建Memorit实例
	s := raft.NewMemoryStorage()
	if snapshot != nil {
		// 快照数据记录到 Memortstorage 实例
		s.ApplySnapshot(*snapshot)
	}
	s.SetHardState(st)
	// 追加entry记录
	s.Append(ents)
	// 创建 raft 配置实例
	c := &raft.Config{
		ID:              uint64(id),
		ElectionTick:    cfg.ElectionTicks,
		HeartbeatTick:   1,
		Storage:         s,
		MaxSizePerMsg:   maxSizePerMsg,
		MaxInflightMsgs: maxInflightMsgs,
		CheckQuorum:     true,
		PreVote:         cfg.PreVote,
	}
	// 调用 raft 重启
	n := raft.RestartNode(c)
	raftStatusMu.Lock()
	raftStatus = n.Status
	raftStatusMu.Unlock()
	return id, cl, n, s, w
}
```
## 注册 handler
etcdserver.Newserver之后，transport实例已经启动，可以在上边注册handler，用于集群内部个节点通信
```go
func newPeerHandler(
	lg *zap.Logger,
	s etcdserver.Server,
	raftHandler http.Handler,
	leaseHandler http.Handler,
	hashKVHandler http.Handler,
) http.Handler {
	// 请求集群信息的handler
	peerMembersHandler := newPeerMembersHandler(lg, s.Cluster())
	peerMemberPromoteHandler := newPeerMemberPromoteHandler(lg, s)

	mux := http.NewServeMux()
	// 默认handler 404
	mux.HandleFunc("/", http.NotFound)
	mux.Handle(rafthttp.RaftPrefix, raftHandler)
	mux.Handle(rafthttp.RaftPrefix+"/", raftHandler)
	mux.Handle(peerMembersPath, peerMembersHandler)
	mux.Handle(peerMemberPromotePrefix, peerMemberPromoteHandler)
	if leaseHandler != nil {
		mux.Handle(leasehttp.LeasePrefix, leaseHandler)
		mux.Handle(leasehttp.LeaseInternalPrefix, leaseHandler)
	}
	if hashKVHandler != nil {
		mux.Handle(etcdserver.PeerHashKVPath, hashKVHandler)
	}
	mux.HandleFunc(versionPath, versionHandler(s.Cluster(), serveVersion))
	return mux
}
```
## 启动 EtcdServer.Start()
```go
func (s *EtcdServer) Start() {
	// 启动一个协程，执行 EtcdServer.run()
	s.start()
	s.goAttach(func() { s.adjustTicks() })
	// 启动一个后台协程，将当前节点相关信息发送到其他节点（http put）
	s.goAttach(func() { s.publish(s.Cfg.ReqTimeout()) })
	// 清理 wal 日志和快照文件
	s.goAttach(s.purgeFile)
	// 监控相关
	s.goAttach(func() { monitorFileDescriptor(s.getLogger(), s.stopping) })
	// 监控其他节点版本信息，主要是升级场景
	s.goAttach(s.monitorVersions)
	// 线性读
	s.goAttach(s.linearizableReadLoop)
	// 监控 kv hash 相关
	s.goAttach(s.monitorKVHash)
}
```

### start
完成 etcdServer 剩余初始化字段，然后执行 run（启动核心）
```go
func (s *EtcdServer) start() {
	...
	go s.run()
}
```
run
```go
func (s *EtcdServer) run() {
	lg := s.getLogger()
	sn, err := s.r.raftStorage.Snapshot()
	// asynchronously accept apply packets, dispatch progress in-order
	// 启动 fifo 调用器
	sched := schedule.NewFIFOScheduler()

	var (
		smu   sync.RWMutex
		syncC <-chan time.Time
	)
	// 用来设置 sync 消息定时器
	setSyncC := func(ch <-chan time.Time) {
		smu.Lock()
		syncC = ch
		smu.Unlock()
	}
	getSyncC := func() (ch <-chan time.Time) {
		smu.RLock()
		ch = syncC
		smu.RUnlock()
		return
	}
	// 实例化 raftReadyHandler
	rh := &raftReadyHandler{
		getLead:    func() (lead uint64) { return s.getLead() },
		updateLead: func(lead uint64) { s.setLead(lead) },
		// leader 发生变化的一些操作
		updateLeadership: func(newLeader bool) {
			if !s.isLeader() {
				if s.lessor != nil {  // leasor 降级
					s.lessor.Demote()
				}
				if s.compactor != nil {
					s.compactor.Pause()
				}
				setSyncC(nil)  // 非leader 节点不发送 sync 消息
			} else {
				if newLeader {
					t := time.Now()
					s.leadTimeMu.Lock()
					s.leadElectedTime = t
					s.leadTimeMu.Unlock()
				}
				// leader 定期发送 sync 消息
				setSyncC(s.SyncTicker.C)
				if s.compactor != nil {
					s.compactor.Resume()
				}
			}
			if newLeader {
				s.leaderChangedMu.Lock()
				lc := s.leaderChanged
				s.leaderChanged = make(chan struct{})
				close(lc)
				s.leaderChangedMu.Unlock()
			}
			// TODO: remove the nil checking
			// current test utility does not provide the stats
			if s.stats != nil {
				s.stats.BecomeLeader()
			}
		},
		// 更新 Server 的 committedIndex
		updateCommittedIndex: func(ci uint64) {
			cci := s.getCommittedIndex()
			if ci > cci {
				s.setCommittedIndex(ci)
			}
		},
	}
	// 启动raft，会启动协程处理 Ready 实例
	s.r.start(rh)

	ep := etcdProgress{
		confState: sn.Metadata.ConfState,
		snapi:     sn.Metadata.Index,
		appliedt:  sn.Metadata.Term,
		appliedi:  sn.Metadata.Index,
	}

	defer func() {
      ...
	}()

	var expiredLeaseC <-chan []*lease.Lease
	if s.lessor != nil {
		expiredLeaseC = s.lessor.ExpiredLeasesC()
	}

	for {
		select {
		case ap := <-s.r.apply():  // 读取 raftNode 的applyC通道 并进行处理, 后边介绍 applyAll()
			f := func(context.Context) { s.applyAll(&ep, &ap) }
			// fifo 调度
			sched.Schedule(f)
			// 有过期的租约，后边说明
		case leases := <-expiredLeaseC:
			
		// 定时发送 sync 消息
		case <-getSyncC():
			if s.v2store.HasTTLKeys() {
				s.sync(s.Cfg.ReqTimeout())
			}
		case <-s.stop:
			return
		}
	}
}
```

### 处理 APPly
```go
func (s *EtcdServer) applyAll(ep *etcdProgress, apply *apply) {
	// 处理快照数据
	s.applySnapshot(ep, apply)
	// 处理添加日志
	s.applyEntries(ep, apply)

	proposalsApplied.Set(float64(ep.appliedi))
	s.applyWait.Trigger(ep.appliedi)

    //  根据上边处理的结果检测是否需要生成新的快照，最后处理 MSGSNAP消息
	// wait for the raft routine to finish the disk writes before triggering a
	// snapshot. or applied index might be greater than the last index in raft
	// storage, since the raft routine might be slower than apply routine.
	<-apply.notifyc

	s.triggerSnapshot(ep)
	select {
	// snapshot requested via send()
	case m := <-s.r.msgSnapC:
		merged := s.createMergedSnapshotMessage(m, ep.appliedt, ep.appliedi, ep.confState)
		s.sendMergedSnap(merged)
	default:
	}
}
```