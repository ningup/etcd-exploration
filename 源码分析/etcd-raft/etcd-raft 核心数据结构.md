# raft.raft
raft 算法的核心数据结构，封装了一个raft节点所有信息
```go
type raft struct {
	// 节点 id
	id uint64

    // 当前节点的任期号
	Term uint64
	// 当前任期该节点投票的给的节点
	Vote uint64

    // 只读请求相关
	readStates []ReadState

	// the log
	raftLog *raftLog

	maxMsgSize         uint64
	maxUncommittedSize uint64
	// TODO(tbg): rename to trk.
	prs tracker.ProgressTracker

    // 当前角色
	state StateType

	// isLearner is true if the local raft node is a learner.
	isLearner bool

    // 待发送的消息列表，上层应用负责发送
	msgs []pb.Message

	// the leader id
	lead uint64
	// leadTransferee is id of the leader transfer target when its value is not zero.
	// Follow the procedure defined in raft thesis 3.10.
	leadTransferee uint64
	// Only one conf change may be pending (in the log, but not yet
	// applied) at a time. This is enforced via pendingConfIndex, which
	// is set to a value >= the log index of the latest pending
	// configuration change (if any). Config changes are only allowed to
	// be proposed if the leader's applied index is greater than this
	// value.
	pendingConfIndex uint64
	// an estimate of the size of the uncommitted tail of the Raft log. Used to
	// prevent unbounded log growth. Only maintained by the leader. Reset on
	// term changes.
	uncommittedSize uint64

	readOnly *readOnly

	// number of ticks since it reached last electionTimeout when it is leader
	// or candidate.
	// number of ticks since it reached last electionTimeout or received a
	// valid message from current leader when it is a follower.
	// 选举计时器
	electionElapsed int

	// number of ticks since it reached last heartbeatTimeout.
	// only leader keeps heartbeatElapsed.
	// 心跳计时器
	heartbeatElapsed int

    // 下边两个字段用于网络分区场景做的优化，防止出现多个leader时的消耗
	checkQuorum bool
	preVote     bool

    // 两个计时器的超时时间
	heartbeatTimeout int
	electionTimeout  int
	// randomizedElectionTimeout is a random number between
	// [electiontimeout, 2 * electiontimeout - 1]. It gets reset
	// when raft changes its state to follower or candidate.
	randomizedElectionTimeout int
	disableProposalForwarding bool

    // 推进逻辑时钟的函数，不同角色对应的方法不一样，如tickHeartbeat(), tickElection()
	tick func()
	// 当前节点收到消息时的处理函数，不同角色对应方法不同，stepLeader(),stepFollower() 等
	step stepFunc

	logger Logger
}
```

# raft.Config
主要用于配置参数的传递，实例化raft的时候需要这个配置结构
```go
type Config struct {
	// ID is the identity of the local raft. ID cannot be 0.
	ID uint64
	peers []uint64
	learners []uint64

    // 用于修改两个计时器的超时时间的
	ElectionTick int
	HeartbeatTick int
	// 用于保存节点 raft 产生的日志记录，上层应用传递进来的，后边会详细说明
	Storage Storage
	// 当前应用的 index，重启之后需要设置
	Applied uint64 for unlimited,
	// 0 for at most one entry per message.
	MaxSizePerMsg uint64
	// MaxCommittedSizePerReady limits the size of the committed entries which
	// can be applied.
	MaxCommittedSizePerReady uint64
	// MaxUncommittedEntriesSize limits the aggregate byte size of the
	// uncommitted entries that may be appended to a leader's log. Once this
	// limit is exceeded, proposals will begin to return ErrProposalDropped
	// errors. Note: 0 for no limit.
	MaxUncommittedEntriesSize uint64
    // 已经发送出去但是未响应的消息最大个数
	MaxInflightMsgs int

    // 上边介绍过
	CheckQuorum bool
	PreVote bool

    // 只读请求相关
	ReadOnlyOption ReadOnlyOption

	// Logger is the logger used for raft log. For multinode which can host
	// multiple raft group, each raft group can have its own logger
	Logger Logger

	// DisableProposalForwarding set to true means that followers will drop
	// proposals, rather than forwarding them to the leader. One use case for
	// this feature would be in a situation where the Raft leader is used to
	// compute the data of a proposal, for example, adding a timestamp from a
	// hybrid logical clock to data in a monotonically increasing way. Forwarding
	// should be disabled to prevent a follower with an inaccurate hybrid
	// logical clock from assigning the timestamp and then forwarding the data
	// to the leader.
	DisableProposalForwarding bool
}
```

# raft.Storage
raft不负责持久化数据存储和网络通信，网络数据都是通过Node接口的函数传入和传出raft。持久化数据存储由创建raft.Node的应用层负责，包含：
* 应用层使用Entry生成的状态机，即一致的应用数据。
* WAL： 历史的Entry（包含还未达成一致的Entry）和快照数据。
* Snapshot是已在节点间达成一致Entry的快照，快照之前的Entry必然都是已经达成一致的，而快照之后，有达成一致的，也有写入磁盘还未达成一致的Entry。
* raft会使用到这些Entry和快照，Storage接口，就是用来读这些数据的。

```go
type Storage interface {
	// TODO(tbg): split this into two interfaces, LogStorage and StateStorage.

	// InitialState returns the saved HardState and ConfState information.
	InitialState() (pb.HardState, pb.ConfState, error)
	// Entries returns a slice of log entries in the range [lo,hi).
	// MaxSize limits the total size of the log entries returned, but
	// Entries returns at least one entry if any.
	// 获取指定 index的 entry切片
	Entries(lo, hi, maxSize uint64) ([]pb.Entry, error)
	// Term returns the term of entry i, which must be in the range
	// [FirstIndex()-1, LastIndex()]. The term of the entry before
	// FirstIndex is retained for matching purposes even though the
	// rest of that entry may not be available.
	// 获取某个entry所在的任期
	Term(i uint64) (uint64, error)
	// LastIndex returns the index of the last entry in the log.
	// 最新的 entry 索引
	LastIndex() (uint64, error)
	// FirstIndex returns the index of the first log entry that is
	// possibly available via Entries (older entries have been incorporated
	// into the latest Snapshot; if storage only contains the dummy entry the
	// first log entry is not available).
	// 获取本节点第一个 entry 索引，snapshot后边的第一个
	FirstIndex() (uint64, error)
	// Snapshot returns the most recent snapshot.
	// If snapshot is temporarily unavailable, it should return ErrSnapshotTemporarilyUnavailable,
	// so raft state machine could know that Storage needs some time to prepare
	// snapshot and call Snapshot later.
	// 获取本节点最近生成的Snapshot，Snapshot是由应用层创建的，并暂时保存起来，raft调用此接口读取
	Snapshot() (pb.Snapshot, error)
}
```

# raft.MemoryStorage
每次都从磁盘文件读取这些数据，效率必然是不高的，所以etcd/raft内定义了MemoryStorage，它实现了Storage接口，并且提供了函数来维护最新快照后的Entry
```go
type MemoryStorage struct {
	// Protects access to all fields. Most methods of MemoryStorage are
	// run on the raft goroutine, but Append() is run on an application
	// goroutine.
	sync.Mutex

	hardState pb.HardState
	snapshot  pb.Snapshot
	// ents[i] has raft log position i+snapshot.Metadata.Index
	// 快照之后的 entry
	ents []pb.Entry
}
```

# raft.unstable
由于 Entry 的存储是由应用层负责的，所以raft需要暂时存储还未存到Storage中的Entry或者Snapshot，在创建Ready时，Entry和Snapshot会被封装到Ready，由应用层写入到storage。
```go
type unstable struct {
	// the incoming unstable snapshot, if any.
	// folloer 刚发过来还没存储的 snapshot
	snapshot *pb.Snapshot
	// all entries that have not yet been written to storage.
    // entries：对leader而已，是raft刚利用请求创建的Entry，对follower而言是从leader收到的Entry。
	entries []pb.Entry
    // offset：Entries[i].Index = i + offset。
	offset  uint64

	logger Logger
}
```

# raft.raftLog
raft使用raftLog来管理当前Entry序列和Snapshot等信息。
```go
type raftLog struct {
	// storage contains all stable entries since the last snapshot.
	storage Storage

	// unstable contains all unstable entries and snapshot.
	// they will be saved into storage.
	unstable unstable

	// committed is the highest log position that is known to be in
	// stable storage on a quorum of nodes.
	// 最后一个在raft集群多数节点之间达成一致的Entry Index。
	committed uint64
	// applied is the highest log position that the application has
	// been instructed to apply to its state machine.
	// Invariant: applied <= committed
	// 当前节点被应用层应用到状态机的最后一个Entry Index。applied和committed之间的Entry就是等待被应用层应用到状态机的Entry。
	applied uint64

	logger Logger

	// maxNextEntsSize is the maximum number aggregate byte size of the messages
	// returned from calls to nextEnts.
	maxNextEntsSize uint64
}
```
![](../img/8.png)

# raft.SoftState
```go
type SoftState struct {
	// leader id
	Lead      uint64 // must use atomic operations to access; keep 64-bit aligned.
	// 节点角色
	RaftState StateType
}
```
# raftpb.HardState
```go
type HardState struct {
	// 当前节点的Term
	Term             uint64 `protobuf:"varint,1,opt,name=term" json:"term"`
	// 投票给的节点id
	Vote             uint64 `protobuf:"varint,2,opt,name=vote" json:"vote"`
	// 当前 committed 的 entry index
	Commit           uint64 `protobuf:"varint,3,opt,name=commit" json:"commit"`
	XXX_unrecognized []byte `json:"-"`
}
```
会写入到 wal，节点重启会回复

# raftpb.Entry
每个Raft集群节点都是一个状态机，每个节点都使用相同的log entry序列修改状态机的数据，Entry就是每一个操作项，raft的核心能力就是为应用层提供序列相同的entry。
```go
type Entry struct {
	// 任期
	Term             uint64    `protobuf:"varint,2,opt,name=Term" json:"Term"`
	// 每一个Entry都有一个的Index，代表当前Entry在log entry序列中的位置，每个index上最终只有1个达成共识的Entry。
	Index            uint64    `protobuf:"varint,3,opt,name=Index" json:"Index"`
	// 表明当前Entry的类型，EntryNormal/EntryConfChange/EntryConfChangeV2
	Type             EntryType `protobuf:"varint,1,opt,name=Type,enum=raftpb.EntryType" json:"Type"`
	// 序列化的数据
	Data             []byte    `protobuf:"bytes,4,opt,name=Data" json:"Data,omitempty"`
	XXX_unrecognized []byte    `json:"-"`
}
```

# raftpb.snapshot
* 历史的 entry 已经没有意义

```go
type Snapshot struct {
	Data             []byte           `protobuf:"bytes,1,opt,name=data" json:"data,omitempty"`
	Metadata         SnapshotMetadata `protobuf:"bytes,2,opt,name=metadata" json:"metadata"`
	XXX_unrecognized []byte           `json:"-"`
}
```

# raftpb.SnapshotMetadata
快照的元数据
```go
type SnapshotMetadata struct {
	// raft 的配置状态
	ConfState        ConfState `protobuf:"bytes,1,opt,name=conf_state,json=confState" json:"conf_state"`
	// 快照对应的 entry 的任期和索引
	Index            uint64    `protobuf:"varint,2,opt,name=index" json:"index"`
	Term             uint64    `protobuf:"varint,3,opt,name=term" json:"term"`
	XXX_unrecognized []byte    `json:"-"`
}
```

# raftpb.Message
节点间通讯的结构体
```go
type Message struct {
	Type             MessageType `protobuf:"varint,1,opt,name=type,enum=raftpb.MessageType" json:"type"`
	To               uint64      `protobuf:"varint,2,opt,name=to" json:"to"`
	From             uint64      `protobuf:"varint,3,opt,name=from" json:"from"`
	Term             uint64      `protobuf:"varint,4,opt,name=term" json:"term"`
	// 创建Message时，发送节点本地所保存的log entry序列中最大的Term，在选举的时候会使用。
	LogTerm          uint64      `protobuf:"varint,5,opt,name=logTerm" json:"logTerm"`
	// 不同的消息类型，Index的含义不同。Term和Index与Entry中的Term和Index不一定会相同，因为某个follower可能比较慢，leader向follower发送已经committed的Entry。
	Index            uint64      `protobuf:"varint,6,opt,name=index" json:"index"`
	// 发送给 follower 待处理的 entries
	Entries          []Entry     `protobuf:"bytes,7,rep,name=entries" json:"entries"`
	// 不同消息含义不同
	Commit           uint64      `protobuf:"varint,8,opt,name=commit" json:"commit"`
	// leader 传递给folloer 的 snapshot
	Snapshot         Snapshot    `protobuf:"bytes,9,opt,name=snapshot" json:"snapshot"`
	Reject           bool        `protobuf:"varint,10,opt,name=reject" json:"reject"`
	RejectHint       uint64      `protobuf:"varint,11,opt,name=rejectHint" json:"rejectHint"`
	Context          []byte      `protobuf:"bytes,12,opt,name=context" json:"context,omitempty"`
	XXX_unrecognized []byte      `json:"-"`
}
```

# raft.Node
* raft 本身没有实现网络，持久化，也没有提供友好api
* Node 是 raft 上边的一层封装，对外提供建议 API

```go
type Node interface {
	// Tick increments the internal logical clock for the Node by a single tick. Election
	// timeouts and heartbeat timeouts are in units of ticks.
	// 推进逻辑时钟的方法
	Tick()
	// Campaign causes the Node to transition to candidate state and start campaigning to become leader.
	// 切换角色状态
	Campaign(ctx context.Context) error
	// Propose proposes that data be appended to the log. Note that proposals can be lost without
	// notice, therefore it is user's job to ensure proposal retries.
	// 把写请求封装成MsgProp消息
	Propose(ctx context.Context, data []byte) error
	// ProposeConfChange proposes a configuration change. Like any proposal, the
	// configuration change may be dropped with or without an error being
	// returned. In particular, configuration changes are dropped unless the
	// leader has certainty that there is no prior unapplied configuration
	// change in its log.
	//
	// The method accepts either a pb.ConfChange (deprecated) or pb.ConfChangeV2
	// message. The latter allows arbitrary configuration changes via joint
	// consensus, notably including replacing a voter. Passing a ConfChangeV2
	// message is only allowed if all Nodes participating in the cluster run a
	// version of this library aware of the V2 API. See pb.ConfChangeV2 for
	// usage details and semantics.
	// 修改集群状态的消息封装
	ProposeConfChange(ctx context.Context, cc pb.ConfChangeI) error

	// Step advances the state machine using the given message. ctx.Err() will be returned, if any.
	// 收到消息后，交给底层raft处理
	Step(ctx context.Context, msg pb.Message) error

	// Ready returns a channel that returns the current point-in-time state.
	// Users of the Node must call Advance after retrieving the state returned by Ready.
	//
	// NOTE: No committed entries from the next Ready may be applied until all committed entries
	// and snapshots from the previous one have finished.
	// 与上层应用进行数据交互的通道
	Ready() <-chan Ready

	// Advance notifies the Node that the application has saved progress up to the last Ready.
	// It prepares the node to return the next available Ready.
	//
	// The application should generally call Advance after it applies the entries in last Ready.
	//
	// However, as an optimization, the application may call Advance while it is applying the
	// commands. For example. when the last Ready contains a snapshot, the application might take
	// a long time to apply the snapshot data. To continue receiving Ready without blocking raft
	// progress, it can call Advance before finishing applying the last ready.
	// 处理完ready后调用该方法告知底层raft
	Advance()
	// ApplyConfChange applies a config change (previously passed to
	// ProposeConfChange) to the node. This must be called whenever a config
	// change is observed in Ready.CommittedEntries.
	//
	// Returns an opaque non-nil ConfState protobuf which must be recorded in
	// snapshots.
	ApplyConfChange(cc pb.ConfChangeI) *pb.ConfState

	// TransferLeadership attempts to transfer leadership to the given transferee.
	TransferLeadership(ctx context.Context, lead, transferee uint64)

	// ReadIndex request a read state. The read state will be set in the ready.
	// Read state has a read index. Once the application advances further than the read
	// index, any linearizable read requests issued before the read request can be
	// processed safely. The read state will have the same rctx attached.
	// 处理只读请求
	ReadIndex(ctx context.Context, rctx []byte) error

	// Status returns the current status of the raft state machine.
	Status() Status
	// ReportUnreachable reports the given node is not reachable for the last send.
	ReportUnreachable(id uint64)
	// ReportSnapshot reports the status of the sent snapshot. The id is the raft ID of the follower
	// who is meant to receive the snapshot, and the status is SnapshotFinish or SnapshotFailure.
	// Calling ReportSnapshot with SnapshotFinish is a no-op. But, any failure in applying a
	// snapshot (for e.g., while streaming it from leader to follower), should be reported to the
	// leader with SnapshotFailure. When leader sends a snapshot to a follower, it pauses any raft
	// log probes until the follower can apply the snapshot and advance its state. If the follower
	// can't do that, for e.g., due to a crash, it could end up in a limbo, never getting any
	// updates from the leader. Therefore, it is crucial that the application ensures that any
	// failure in snapshot sending is caught and reported back to the leader; so it can resume raft
	// log probing in the follower.
	// 通知底层 raft 上次发送快照的结果
	ReportSnapshot(id uint64, status SnapshotStatus)
	// Stop performs any necessary termination of the Node.
	// 关闭当前节点
	Stop()
}
```
# raft.node
实现node接口，持有一些列通道
```go
type node struct {
	propc      chan msgWithResult
	recvc      chan pb.Message
	confc      chan pb.ConfChangeV2
	confstatec chan pb.ConfState
	readyc     chan Ready
	advancec   chan struct{}
	tickc      chan struct{}
	done       chan struct{}
	stop       chan struct{}
	status     chan chan Status

	rn *RawNode
}
```

# raft.Ready
raft 使用 Ready 对外传递数据，上层引用处理完一个后调用 Advance(), 通知 raft 产生下一个 ready
```go
type Ready struct {
	// The current volatile state of a Node.
	// SoftState will be nil if there is no update.
	// It is not required to consume or store SoftState.
	*SoftState

	// The current state of a Node to be saved to stable storage BEFORE
	// Messages are sent.
	// HardState will be equal to empty state if there is no update.
	pb.HardState

	// ReadStates can be used for node to serve linearizable read requests locally
	// when its applied index is greater than the index in ReadState.
	// Note that the readState will be returned when raft receives msgReadIndex.
	// The returned is only valid for the request that requested to read.
	ReadStates []ReadState

	// Entries specifies entries to be saved to stable storage BEFORE
	// Messages are sent.
	// Entries保存的是从unstable读取的Entry，它们即将被应用层写入storage
	Entries []pb.Entry

	// Snapshot specifies the snapshot to be saved to stable storage.
	Snapshot pb.Snapshot

	// CommittedEntries specifies entries to be committed to a
	// store/state-machine. These have previously been committed to stable
	// store.
	// 已经被 Committed，还没有applied，应用层会把他们应用到状态机。
	CommittedEntries []pb.Entry

	// Messages specifies outbound messages to be sent AFTER Entries are
	// committed to stable storage.
	// If it contains a MsgSnap message, the application MUST report back to raft
	// when the snapshot has been received or has failed by calling ReportSnapshot.
	// 需要处理的消息
	Messages []pb.Message

	// MustSync indicates whether the HardState and Entries must be synchronously
	// written to disk or if an asynchronous write is permissible.
	MustSync bool
}
```