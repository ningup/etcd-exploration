# 3. 实现
## 3.1 入口 main.go
```go
func main() {
    //------------初始化参数：cluster,id,kvport,join-------------//

    //--------------建立应用层和raft之间的通信通道 --------------//
    proposeC := make(chan string)
    defer close(proposeC)
    confChangeC := make(chan raftpb.ConfChange)
    defer close(confChangeC)

    // raft provides a commit stream for the proposals from the http api
    // ---------------启动 raft ------------------------//
    var kvs *kvstore
    getSnapshot := func() ([]byte, error) { return kvs.getSnapshot() }
    commitC, errorC, snapshotterReady := newRaftNode(*id, strings.Split(*cluster, ","), *join, getSnapshot, proposeC, confChangeC)

    //--------------------启动 kv存储服务---------------------//
    kvs = newKVStore(<-snapshotterReady, proposeC, commitC, errorC)

    // the key-value http handler will propose updates to raft
    //-------------------启动 http 对外服务-------------------//
    serveHttpKVAPI(kvs, *kvport, confChangeC, errorC)
}
```

## 3.2 httpKVAPI
数据结构
```go
type httpKVAPI struct {
    //  kvstore 实例
    store       *kvstore
    // 向 raft 提交集群配置变更的通道
	confChangeC chan<- raftpb.ConfChange
}
```

http 服务实现
```go
// serveHttpKVAPI starts a key-value server with a GET/PUT API and listens.
func serveHttpKVAPI(kv *kvstore, port int, confChangeC chan<- raftpb.ConfChange, errorC <-chan error) {
	srv := http.Server{
        Addr: ":" + strconv.Itoa(port),
        // 设置 handler
		Handler: &httpKVAPI{
			store:       kv,
			confChangeC: confChangeC,
		},
    }
    // 启动一个goroutine监听服务，有http请求时，httpserver 会单独启动新的 goroutine进行处理
	go func() {
		if err := srv.ListenAndServe(); err != nil {
			log.Fatal(err)
		}
    }()
    
    // exit when raft goes down
    // 阻塞，直至 errorC 有内容
	if err, ok := <-errorC; ok {
		log.Fatal(err)
	}
}
```

以下是各类请求的 handler

* PUT 请求，向 kvstore 更新键值数据
```go
func (h *httpKVAPI) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 获取 url 中的 key
	key := r.RequestURI
	switch {
    case r.Method == "PUT":
        // 读取请求体和读取过程中的错误
		v, err := ioutil.ReadAll(r.Body)
        // 将键值对序列化并写入 proposeC 通道中 
		h.store.Propose(key, string(v))

		// Optimistic-- no waiting for ack from raft. Value is not yet
        // committed so a subsequent GET on the key may return old value
        // 返回状态码
		w.WriteHeader(http.StatusNoContent)
}
```
* Get 请求，从 kvsore 实例中读取指定键值对
```go
func (h *httpKVAPI) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 获取 url 中的 key
	key := r.RequestURI
	switch {
    case r.Method == "GET":
        // 从 kvstore 直接返回
		if v, ok := h.store.Lookup(key); ok {
			w.Write([]byte(v))
		} else {
            // 404
			http.Error(w, "Failed to GET", http.StatusNotFound)
		}
}
```
* POST 请求，向集群提交增加节点
```go
func (h *httpKVAPI) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 获取 url 作为 key
	key := r.RequestURI
    case r.Method == "POST":
        // 获取新加节点的 url
		url, err := ioutil.ReadAll(r.Body)
        // 解析节点id
        nodeId, err := strconv.ParseUint(key[1:], 0, 64)
        // 构造 ConfChange 实例
		cc := raftpb.ConfChange{
			Type:    raftpb.ConfChangeAddNode,  // 表示新增节点
			NodeID:  nodeId,
			Context: url,
        }
        // 把实例提交给 confChangeC 通道
		h.confChangeC <- cc

		// As above, optimistic that raft will apply the conf change
		w.WriteHeader(http.StatusNoContent)
}
```
* delete 请求是 post 请求的逆向

## 3.3 kvstore
### 数据结构
```go
// a key-value store backed by raft
type kvstore struct {
	proposeC    chan<- string // channel for proposing updates
	mu          sync.RWMutex
	kvStore     map[string]string // current committed key-value pairs
	snapshotter *snap.Snapshotter
}
```
* proposeC 同上处理更新请求的通道
* kvStore 存储当前已经提交的键值对 map
* snapshotter 负责读取快照文件

### 初始化 kvstore
```go
func newKVStore(snapshotter *snap.Snapshotter, proposeC chan<- string, commitC <-chan *string, errorC <-chan error) *kvstore {
	s := &kvstore{proposeC: proposeC, kvStore: make(map[string]string), snapshotter: snapshotter}
    // replay log into key-value map
    // 先读取一下 log
	s.readCommits(commitC, errorC)
    // read commits from raft into kvStore map until error
    // 再后台启动一个协程持续监听 commitC 通道
	go s.readCommits(commitC, errorC)
	return s
}
```
读取 commitC 通道具体实现逻辑如下：
```go
func (s *kvstore) readCommits(commitC <-chan *string, errorC <-chan error) {
    // 循环读取通道
	for data := range commitC {
        // 说明需要读取快照数据
		if data == nil {
            // 读取快照
            snapshot, err := s.snapshotter.Load()
            // 加载快照到 map 里
			if err := s.recoverFromSnapshot(snapshot.Data); 
			continue
		}

        // 将读取到的 kv 数据反序列化得到kv实例
		var dataKv kv
		dec := gob.NewDecoder(bytes.NewBufferString(*data))
        s.mu.Lock()
        // 保存到 kvstore 里
		s.kvStore[dataKv.Key] = dataKv.Val
		s.mu.Unlock()
	}
}
```

请求 GET key 时的方法：
```go
func (s *kvstore) Lookup(key string) (string, bool) {
	s.mu.RLock()
    defer s.mu.RUnlock()
    // 直接读取 map
	v, ok := s.kvStore[key]
	return v, ok
}
```

## 3.4 raftNode
raftNode 是对 etcd-raft 的一层封装，主要提供的功能如下：
* 将客户端发来的请求传递给 etcd-raft 进行处理
* 从 etcd-raft 的 node.readyc 通道中解析 Ready 实例，并进行处理
* 管理 wal 日志文件
* 管理快照数据
* 管理逻辑时钟
* 将 etcd-raft 返回的待发送消息通过网络组件发送到其他节点中

### 3.4.1 raftNode 核心字段
```go
// A key-value stream backed by raft
type raftNode struct {
	proposeC    <-chan string            // proposed messages (k,v)
	confChangeC <-chan raftpb.ConfChange // proposed cluster config changes
	commitC     chan<- *string           // entries committed to log (k,v)
	errorC      chan<- error             // errors from raft session

	id          int      // client ID for raft session
	peers       []string // raft peer URLs
	join        bool     // node is joining an existing cluster
	waldir      string   // path to WAL directory
	snapdir     string   // path to snapshot directory
	getSnapshot func() ([]byte, error)
	lastIndex   uint64 // index of log at start

	confState     raftpb.ConfState
	snapshotIndex uint64
	appliedIndex  uint64

	// raft backing for the commit/error channel
	node        raft.Node
	raftStorage *raft.MemoryStorage
	wal         *wal.WAL

	snapshotter      *snap.Snapshotter
	snapshotterReady chan *snap.Snapshotter // signals when snapshotter is ready

    snapCount uint64
    // 用于节点通讯实例
	transport *rafthttp.Transport
	stopc     chan struct{} // signals proposal channel closed
	httpstopc chan struct{} // signals http server to shutdown
	httpdonec chan struct{} // signals http server shutdown complete
}
```

### 3.4.2 初始化 raftNode
```go
func newRaftNode(id int, peers []string, join bool, getSnapshot func() ([]byte, error), proposeC <-chan string,
	confChangeC <-chan raftpb.ConfChange) (<-chan *string, <-chan error, <-chan *snap.Snapshotter) {

	commitC := make(chan *string)
	errorC := make(chan error)

	rc := &raftNode{
		proposeC:    proposeC,
		confChangeC: confChangeC,
		commitC:     commitC,
		errorC:      errorC,
		id:          id,
		peers:       peers,
        join:        join,
        // 初始化 存放 wal 和 snap文件的目录
		waldir:      fmt.Sprintf("raftexample-%d", id),
		snapdir:     fmt.Sprintf("raftexample-%d-snap", id),
		getSnapshot: getSnapshot,
		snapCount:   defaultSnapshotCount,
		stopc:       make(chan struct{}),
		httpstopc:   make(chan struct{}),
		httpdonec:   make(chan struct{}),

		snapshotterReady: make(chan *snap.Snapshotter, 1),
		// rest of structure populated after WAL replay
    }
    // 单独启动一个协程完成剩余初始化工作
	go rc.startRaft()
	return commitC, errorC, rc.snapshotterReady
}
```
主要工作
* 创建通道，初始化相关字段，注意 rest of structure populated after WAL replay
* 调用 startRaft 进行剩余的初始化工作
* 返回各类通道 

startRaft() 实现：
```go
func (rc *raftNode) startRaft() {
    // 创建 snapshotter 实例
    rc.snapshotter = snap.New(zap.NewExample(), rc.snapdir)
    // 实例返回给上层模块
	rc.snapshotterReady <- rc.snapshotter

    // 创建 wal 实例，加载快照并回放 wal 日志
    oldwal := wal.Exist(rc.waldir)
    /*
        1. 读取快照文件记录的最后一条 Entry 记录、TERM和索引值
        2. 根据索引值等信息读取 WAL 日志文件的位置并进行日志的读取
        3. 最后将快照的位置以及WAL日志记录等信息记录到 raftStorage 
        4. 详情见后边 WAL 章节详解
    */
	rc.wal = rc.replayWAL()

    // 创建 raft 配置实例 config
	c := &raft.Config{
		ID:                        uint64(rc.id),
		ElectionTick:              10,
		HeartbeatTick:             1,
		Storage:                   rc.raftStorage,
		MaxSizePerMsg:             1024 * 1024,
		MaxInflightMsgs:           256,
		MaxUncommittedEntriesSize: 1 << 30,
	}

	if oldwal {
		rc.node = raft.RestartNode(c)
	} else {
        // 初始化 etcd-raft 模块
		rc.node = raft.StartNode(c, startPeers)
	}

    // 建立集群见网络连接，并尝试连接
	rc.transport = &rafthttp.Transport{
		Logger:      zap.NewExample(),
		ID:          types.ID(rc.id),
		ClusterID:   0x1000,
		Raft:        rc,
		ServerStats: stats.NewServerStats("", ""),
		LeaderStats: stats.NewLeaderStats(strconv.Itoa(rc.id)),
		ErrorC:      make(chan error),
	}

	rc.transport.Start()
	for i := range rc.peers {
		if i+1 != rc.id {
			rc.transport.AddPeer(types.ID(i+1), []string{rc.peers[i]})
		}
	}

    // 启动协程,用于监听节点网络连接
    go rc.serveRaft()
    // 启动协程，处理上层应用和底层 etcd-raft 之间的交互
	go rc.serveChannels()
}
```
startRaft 里边需要重点看如下几个实现：
* serveRaft 实现：
```go
func (rc *raftNode) serveRaft() {
	url, err := url.Parse(rc.peers[rc.id-1])
    // 与 http.Server一起 用于对当前 url 地址进行监听
	ln, err := newStoppableListener(url.Host, rc.httpstopc)
	if err != nil {
		log.Fatalf("raftexample: Failed to listen rafthttp (%v)", err)
	}
    // 创建 http.Server 实例用上边的 newStoppableListener 监听 url, 后边 raft-http 时会重点描述
    // 需要注意的是Serve 方法会一直阻塞，直至其关闭
	err = (&http.Server{Handler: rc.transport.Handler()}).Serve(ln)
	select {
	case <-rc.httpstopc:
	default:
		log.Fatalf("raftexample: Failed to serve rafthttp (%v)", err)
	}
	close(rc.httpdonec)
}
```
可以看出，主要目的用于监听当前节点的指定端口，处理与其他节点的网络连接

* serveChannels 实现:
```go
func (rc *raftNode) serveChannels() {
    // 根据前边已经回放日志好的信息，读取快照信息
	snap, err := rc.raftStorage.Snapshot()

	rc.confState = snap.Metadata.ConfState
	rc.snapshotIndex = snap.Metadata.Index
	rc.appliedIndex = snap.Metadata.Index

	defer rc.wal.Close()

    // 创建一个定时器，每100ms触发一次，这个是 etcd-raft 最小的时间单位(逻辑时钟)
	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()

	// send proposals over raft
	go func() {
		confChangeCount := uint64(0)

		for rc.proposeC != nil && rc.confChangeC != nil {
			select {
			case prop, ok := <-rc.proposeC:
                // blocks until accepted by raft state machine
                // 通过 raft 的 node 把客户端传过来的数据请求交给 etcd-raft
                rc.node.Propose(context.TODO(), []byte(prop))

            case cc, ok := <-rc.confChangeC:
                // 统计集群变更请求的次数
                confChangeCount++
                cc.ID = confChangeCount
                // 同理， 通过 raft 的 node 把客户端传过来集群变更请求交给 etcd-raft
                rc.node.ProposeConfChange(context.TODO(), cc)
			}
		}
		// client closed channel; shutdown raft if not already
		close(rc.stopc)
	}()

    // event loop on raft state machine updates
    // 下边的循环负责处理底层raft返回来的 ready 数据，
	for {
		select {
        // 推进 raft 逻辑时钟
		case <-ticker.C:
			rc.node.Tick()

		// store raft entries to wal, then publish over commit channel
        case rd := <-rc.node.Ready():
            /*
                1. 将 底层 raft 传过来的状态信息，待持久化的 Entries 等记录到 wal 日志，即使宕机这些信息也可以再下次重启节点后回放日志
            */
            rc.wal.Save(rd.HardState, rd.Entries)
            // 产生了新的快照
			if !raft.IsEmptySnap(rd.Snapshot) {
                // 保存快照
                rc.saveSnap(rd.Snapshot)
                // 新快照写到 Storage 里
                rc.raftStorage.ApplySnapshot(rd.Snapshot)
                // 通知上层应用加载新快照
				rc.publishSnapshot(rd.Snapshot)
            }
            // 将待持久化的 Entries 持久化到 Storage 里
            rc.raftStorage.Append(rd.Entries)
            // 将待发送的消息发送到指定节点
            rc.transport.Send(rd.Messages)
            // 将已经提交待应用的 Entry 记录到上层应用的状态机里
			if ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)); !ok {
				rc.stop()
				return
            }
            // 尝试触发快照
            rc.maybeTriggerSnapshot()
            // 处理完 Ready，通知底层 etcd-raft准备下一个 Ready 实例
			rc.node.Advance()

		case err := <-rc.transport.ErrorC:
			rc.writeError(err)
			return

		case <-rc.stopc:
			rc.stop()
			return
		}
	}
}
```
该函数主要功能如下：
* 第一部分负责监听 proposeC 和 confChangeC 两个通道，将从通道传来的信息传给底层 raft 处理

* 第二部分负责处理底层 raft 返回的数据，这些数据被封装在 Ready 结构体中，包括了已经准备好读取、持久化、提交或者发送给 follower 的 entries 和 messages 等信息

* 配置了一个定时器，定时器到期时调用节点的 Tick() 方法推动 raft 逻辑时钟前进