# 初始化 
## 从 raftexample 初始化开始, startRaft()
1. 读取快照 snapshot，回放日志 wal
2. 创建raftStorage(MemoryStorage)
3. 将快照和日志等信息填充到 raftStorage
4. 创建 raft 需要的 config 对象，里边的 Storage 传位上边的 raftStorage
5. 通过 rc.node = raft.StartNode(c, startPeers)  启动 raft 的 node

## StartNode()
```go
// StartNode returns a new Node given configuration and a list of raft peers.
// It appends a ConfChangeAddNode entry for each given peer to the initial log.
//
// Peers must not be zero length; call RestartNode in that case.
func StartNode(c *Config, peers []Peer) Node {
	if len(peers) == 0 {
		panic("no peers given; use RestartNode instead")
	}
	// 创建原始 raft node，详情见下
	rn, err := NewRawNode(c)
	if err != nil {
		panic(err)
	}
	rn.Bootstrap(peers)

    // 实例化 node，即创建各种 node 结构的通道
	n := newNode(rn)

	go n.run()
	return &n
}
```
* NewRawNode()
```go
func NewRawNode(config *Config) (*RawNode, error) {
	// 初始化 raft
	r := newRaft(config)
	rn := &RawNode{
		raft: r,
	}
	rn.prevSoftSt = r.softState()
	rn.prevHardSt = r.hardState()
	return rn, nil
}
```
## newRaft()
```go
func newRaft(c *Config) *raft {
    // 检查参数合法性
	if err := c.validate(); err != nil {
		panic(err.Error())
    }
    // 创建 raftlog 实例，用于记录 Entry
    raftlog := newLogWithSize(c.Storage, c.Logger, c.MaxCommittedSizePerReady)
    // storage 初始状态是通过本地 Entry 记录回放的
	hs, cs, err := c.Storage.InitialState()
	if err != nil {
		panic(err) // TODO(bdarnell)
	}

	if len(c.peers) > 0 || len(c.learners) > 0 {
		if len(cs.Voters) > 0 || len(cs.Learners) > 0 {
			// TODO(bdarnell): the peers argument is always nil except in
			// tests; the argument should be removed and these tests should be
			// updated to specify their nodes through a snapshot.
			panic("cannot specify both newRaft(peers, learners) and ConfState.(Voters, Learners)")
		}
		cs.Voters = c.peers
		cs.Learners = c.learners
	}

    // 创建 raft 实例
	r := &raft{
		id:                        c.ID,
		lead:                      None,
		isLearner:                 false,
		raftLog:                   raftlog,
		maxMsgSize:                c.MaxSizePerMsg,
		maxUncommittedSize:        c.MaxUncommittedEntriesSize,
		prs:                       tracker.MakeProgressTracker(c.MaxInflightMsgs),
		electionTimeout:           c.ElectionTick,
		heartbeatTimeout:          c.HeartbeatTick,
		logger:                    c.Logger,
		checkQuorum:               c.CheckQuorum,
		preVote:                   c.PreVote,
		readOnly:                  newReadOnly(c.ReadOnlyOption),
		disableProposalForwarding: c.DisableProposalForwarding,
	}

	cfg, prs, err := confchange.Restore(confchange.Changer{
		Tracker:   r.prs,
		LastIndex: raftlog.lastIndex(),
	}, cs)
	if err != nil {
		panic(err)
	}
	assertConfStatesEquivalent(r.logger, cs, r.switchToConfig(cfg, prs))

	if !IsEmptyHardState(hs) {
		r.loadState(hs)
    }
    // 如果 conf 配置了 applied，就重置 raftlog 的 apply的值
	if c.Applied > 0 {
		raftlog.appliedTo(c.Applied)
    }
    // 切换成 follower 状态
	r.becomeFollower(r.Term, None)

	var nodesStrs []string
	for _, n := range r.prs.VoterNodes() {
		nodesStrs = append(nodesStrs, fmt.Sprintf("%x", n))
	}

	r.logger.Infof("newRaft %x [peers: [%s], term: %d, commit: %d, applied: %d, lastindex: %d, lastterm: %d]",
		r.id, strings.Join(nodesStrs, ","), r.Term, r.raftLog.committed, r.raftLog.applied, r.raftLog.lastIndex(), r.raftLog.lastTerm())
	return r
}
```

## n.run()
用于处理 node 对象的各类通道，一个后台go协程
```go
func (n *node) run() {
	var propc chan msgWithResult
	var readyc chan Ready
	var advancec chan struct{}
	var rd Ready

	r := n.rn.raft

	lead := None

	for {
		// 上层模块还没有处理完，因此不需要往 ready 通道写入数据
		if advancec != nil {
			readyc = nil

		} else if n.rn.HasReady() {
            // 本次可以构造 ready
			rd = n.rn.readyWithoutAccept()
			readyc = n.readyc
		}

		select {
        // 读取 propc 通道，获取 MsgPropc 消息，交给 raft.step() 处理
		case pm := <-propc:
			m := pm.m
			m.From = r.id
			err := r.Step(m)
        // 读取 recvc通道，获取非 MsgPropc 消息类型，交给 raft.step()  处理
		case m := <-n.recvc:
			// filter out response message from unknown From.
			if pr := r.prs.Progress[m.From]; pr != nil || !IsResponseMsg(m.Type) {
				r.Step(m)
			}
        // 读取 ConfChange 实例 进行处理
		case cc := <-n.confc:
		   ...
        // 逻辑时钟推进一次，调用 raft.tick() 进行时钟推进
		case <-n.tickc:
			n.rn.Tick()
        // 将创建好的 Ready 对象写入 readyc 通道，等待上层使用
		case readyc <- rd:
			n.rn.acceptReady(rd)
			advancec = n.advancec
        // 上层模块处理完 Ready 实例的信号
		case <-advancec:
			n.rn.Advance(rd)
			rd = Ready{}
			advancec = nil
		case c := <-n.status:
			c <- getStatus(r)
		case <-n.stop:
			close(n.done)
			return
		}
	}
}
```

# 状态切换
继上边的newRfat对象之后，调用了 r.becomeFollower(r.Term, None)  切换成 follower 状态

## becomeFollower()
主要工作是设置 raft 各类变量
```go
func (r *raft) becomeFollower(term uint64, lead uint64) {
	// 处理消息的函数指针
	r.step = stepFollower
	r.reset(term)
	// 推进时钟计时器方法为 竞选计时器
	r.tick = r.tickElection
	r.lead = lead
	r.state = StateFollower
	r.logger.Infof("%x became follower at term %d", r.id, r.Term)
}
```
成为 follower 后，会被上层应用定期触发 tick，推进检测时钟是否超时
```go
func (r *raft) tickElection() {
	r.electionElapsed++
    // 超时
	if r.promotable() && r.pastElectionTimeout() {
		r.electionElapsed = 0
		// 触发选举，后边会介绍各类消息处理流程
		r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
	}
}
```
* becomeCandidate()
当 follower 连接数大于一半时调用 becomeCandidate() 成为候选
```go
func (r *raft) becomeCandidate() {
	// TODO(xiangli) remove the panic when the raft implementation is stable
	if r.state == StateLeader {
		panic("invalid transition [leader -> candidate]")
	}
	r.step = stepCandidate
	r.reset(r.Term + 1)
	r.tick = r.tickElection
	r.Vote = r.id
	r.state = StateCandidate
	r.logger.Infof("%x became candidate at term %d", r.id, r.Term)
}
```

* becomeCandidate()
当 candidate 投票超过半数，则调用 becomeLeader 
```go
func (r *raft) becomeLeader() {
	// TODO(xiangli) remove the panic when the raft implementation is stable
	if r.state == StateFollower {
		panic("invalid transition [follower -> leader]")
	}
	r.step = stepLeader
	r.reset(r.Term)
	r.tick = r.tickHeartbeat
	r.lead = r.id
	r.state = StateLeader

	r.prs.Progress[r.id].BecomeReplicate()

	r.pendingConfIndex = r.raftLog.lastIndex()

	emptyEnt := pb.Entry{Data: nil}
	// 向当前节点追加一条空entry记录
	if !r.appendEntry(emptyEnt) {
		// This won't happen because we just called reset() above.
		r.logger.Panic("empty entry was dropped")
	}

	r.reduceUncommittedSize([]pb.Entry{emptyEnt})
	r.logger.Infof("%x became leader at term %d", r.id, r.Term)
```

* appendEntry()
向当前节点追加 entry
```go
func (r *raft) appendEntry(es ...pb.Entry) (accepted bool) {
	// 最新一条
	li := r.raftLog.lastIndex()
	// 设置entry 的index 和 term
	for i := range es {
		es[i].Term = r.Term
		es[i].Index = li + 1 + uint64(i)
	}

	// 向 raftLog 中 append
	li = r.raftLog.append(es...)
	r.prs.Progress[r.id].MaybeUpdate(li)
	// 尝试提交 entry
	r.maybeCommit()
	return true
}
```

```go
// maybeCommit attempts to advance the commit index. Returns true if
// the commit index changed (in which case the caller should call
// r.bcastAppend).
func (r *raft) maybeCommit() bool {
	mci := r.prs.Committed()
	return r.raftLog.maybeCommit(mci, r.Term)
}
```

# 消息处理
## 消息类型
```go
const (
	// 选举计时器超时创建的消息，触发选举
	MsgHup            MessageType = 0
	MsgBeat           MessageType = 1
	MsgProp           MessageType = 2
	MsgApp            MessageType = 3
	MsgAppResp        MessageType = 4
	MsgVote           MessageType = 5
	MsgVoteResp       MessageType = 6
	MsgSnap           MessageType = 7
	MsgHeartbeat      MessageType = 8
	MsgHeartbeatResp  MessageType = 9
	MsgUnreachable    MessageType = 10
	MsgSnapStatus     MessageType = 11
	MsgCheckQuorum    MessageType = 12
	MsgTransferLeader MessageType = 13
	MsgTimeoutNow     MessageType = 14
	MsgReadIndex      MessageType = 15
	MsgReadIndexResp  MessageType = 16
	MsgPreVote        MessageType = 17
	MsgPreVoteResp    MessageType = 18
)
```
## MsgHup
前边可知，follower 的选举计时器超时会创建 MsgHup 消息并调用 raft.Step()方法处理，该方法是各类消息处理的入口
```go
func (r *raft) Step(m pb.Message) error {
	// 先按照 term 进行分类
	switch {
	case m.Term == 0:
		// local message 本地消息 MsgHup就是本地消息
	case m.Term > r.Term:
           //先不关心
	case m.Term < r.Term:
           //先不关心
	}

	switch m.Type {
	case pb.MsgHup:
	    // 非leader 才会处理
		if r.state != StateLeader {
            // 获取提交但是为应用的 entry
			ents, err := r.raftLog.slice(r.raftLog.applied+1, r.raftLog.committed+1, noLimit)
            // 检测是否有 confchange，有的话放弃选举 
			if n := numOfPendingConf(ents); n != 0 && r.raftLog.committed > r.raftLog.applied {
				r.logger.Warningf("%x cannot campaign at term %d since there are still %d pending configuration changes to apply", r.id, r.Term, n)
				return nil
			}

			r.logger.Infof("%x is starting a new election at term %d", r.id, r.Term)
			// 调用 campaign 进行角色切换
			if r.preVote {
				r.campaign(campaignPreElection)
			} else {
				r.campaign(campaignElection)
			}
		} else {
			r.logger.Debugf("%x ignoring MsgHup because already leader", r.id)
		}
    // 其他先不关心
	return nil
}
```
campaign 除了完成状态切换，也会向其他节点发起同类消息
```go
func (r *raft) campaign(t CampaignType) {
	// 方法最后会发送一条消息
	var term uint64
	var voteMsg pb.MessageType
	if t == campaignPreElection {
         ...
    // 切换成候选
	} else {
		r.becomeCandidate()
		// 想其他节点发起投票消息
		voteMsg = pb.MsgVote
		term = r.Term
	}
	// 统计节点收到的选票，这里考虑的是但节点场景，投票给自己之后就能赢得选举
	if _, _, res := r.poll(r.id, voteRespMsgType(voteMsg), true); res == quorum.VoteWon {
		// We won the election after voting for ourselves (which must mean that
		// this is a single-node cluster). Advance to the next state.
		if t == campaignPreElection {
			r.campaign(campaignElection)
        // 票数足够，成为 leader
		} else {
			r.becomeLeader()
		}
		return
	}
	var ids []uint64
	{
		idMap := r.prs.Voters.IDs()
		ids = make([]uint64, 0, len(idMap))
		for id := range idMap {
			ids = append(ids, id)
		}
		sort.Slice(ids, func(i, j int) bool { return ids[i] < ids[j] })
	}
	for _, id := range ids {
		if id == r.id {
			continue
		}
		r.logger.Infof("%x [logterm: %d, index: %d] sent %s request to %x at term %d",
			r.id, r.raftLog.lastTerm(), r.raftLog.lastIndex(), voteMsg, id, r.Term)

		var ctx []byte
		if t == campaignTransfer {
			ctx = []byte(t)
		}
		// 想其他节点发送消息，主要只是追加到 raft.msg 里, 待上层应用发送
		r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
	}
}
```

## MsgVote
消息处理流程完全相同，会根据策略选择是否投票给发送该消息的节点

## MsgVoteResp
candidate节点会处理该信息，决定是否能成为 leader， 成为leader后广播MsgAPP 消息（或者MsgSnap）
```go
func (r *raft) bcastAppend() {
	r.prs.Visit(func(id uint64, _ *tracker.Progress) {
		if id == r.id {
			return
		}
		r.sendAppend(id)
	})
}
```
查找待发送 entries并发送
```go
func (r *raft) maybeSendAppend(to uint64, sendIfEmpty bool) bool {
	pr := r.prs.Progress[to]
	if pr.IsPaused() {
		return false
	}
	m := pb.Message{}
	m.To = to

	term, errt := r.raftLog.term(pr.Next - 1)
	ents, erre := r.raftLog.entries(pr.Next, r.maxMsgSize)
	if len(ents) == 0 && !sendIfEmpty {
		return false
	}

	if errt != nil || erre != nil { // send snapshot if we failed to get term or entries
		if !pr.RecentActive {
			r.logger.Debugf("ignore sending snapshot to %x since it is not recently active", to)
			return false
		}

		m.Type = pb.MsgSnap
		snapshot, err := r.raftLog.snapshot()
		if err != nil {
			if err == ErrSnapshotTemporarilyUnavailable {
				r.logger.Debugf("%x failed to send snapshot to %x because snapshot is temporarily unavailable", r.id, to)
				return false
			}
			panic(err) // TODO(bdarnell)
		}
		if IsEmptySnap(snapshot) {
			panic("need non-empty snapshot")
		}
		m.Snapshot = snapshot
		sindex, sterm := snapshot.Metadata.Index, snapshot.Metadata.Term
		r.logger.Debugf("%x [firstindex: %d, commit: %d] sent snapshot[index: %d, term: %d] to %x [%s]",
			r.id, r.raftLog.firstIndex(), r.raftLog.committed, sindex, sterm, to, pr)
		pr.BecomeSnapshot(sindex)
		r.logger.Debugf("%x paused sending replication messages to %x [%s]", r.id, to, pr)
	} else {
		m.Type = pb.MsgApp
		m.Index = pr.Next - 1
		m.LogTerm = term
		m.Entries = ents
		m.Commit = r.raftLog.committed
		if n := len(m.Entries); n != 0 {
			switch pr.State {
			// optimistically increase the next when in StateReplicate
			case tracker.StateReplicate:
				last := m.Entries[n-1].Index
				pr.OptimisticUpdate(last)
				pr.Inflights.Add(last)
			case tracker.StateProbe:
				pr.ProbeSent = true
			default:
				r.logger.Panicf("%x is sending append in unhandled state %s", r.id, pr.State)
			}
		}
	}
	r.send(m)
	return true
}
```

## MsgAPP
follower 收到 msgapp 消息后会调用 handleAppendEntries 把日志追加到自己的 raftlog里 
```go
func (r *raft) handleAppendEntries(m pb.Message) {
	if m.Index < r.raftLog.committed {
		r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: r.raftLog.committed})
		return
	}

	if mlastIndex, ok := r.raftLog.maybeAppend(m.Index, m.LogTerm, m.Commit, m.Entries...); ok {
		r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: mlastIndex})
	} else {
		r.logger.Debugf("%x [logterm: %d, index: %d] rejected MsgApp [logterm: %d, index: %d] from %x",
			r.id, r.raftLog.zeroTermOnErrCompacted(r.raftLog.term(m.Index)), m.Index, m.LogTerm, m.Index, m.From)
		r.send(pb.Message{To: m.From, Type: pb.MsgAppResp, Index: m.Index, Reject: true, RejectHint: r.raftLog.lastIndex()})
	}
}
```