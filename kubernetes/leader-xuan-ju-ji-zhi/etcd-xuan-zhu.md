# Etcd选主

## 背景

为了解决单点问题，软件系统工程师引入了数据复制技术，实现多副本。通过数据复制方案，一方面我们可以提高服务可用性，避免单点故障。另一方面，多副本可以提升读吞吐量、甚至就近部署在业务所在的地理位置，降低访问延迟。

### 单点故障

为了解决单点问题，软件系统工程师引入了数据复制技术，实现多副本。通过数据复制方案，一方面我们可以提高服务可用性，避免单点故障。

另一方面，多副本可以提升读吞吐量、甚至就近部署在业务所在的地理位置，降低访问延迟。

### 多副本复制

多副本常用的技术方案主要有主从复制和去中心化复制。主从复制，又分为全同步复制、异步复制、半同步复制，比如 MySQL/Redis 单机主备版就基于主从复制实现的。

跟主从复制相反的就是去中心化复制，它是指在一个 n 副本节点集群中，任意节点都可接受写请求，但一个成功的写入需要 w 个节点确认，读取也必须查询至少 r 个节点。

### etcd写请求

Raft 模块收到提案后，如果当前节点是 Follower，它会转发给 Leader，只有 Leader 才能处理写请求

etcdserver 从 Raft 模块获取到提案Propose后，作为 Leader，它会将 put 提案消息广播给集群各个节点，同时需要把集群 Leader 任期号、投票信息、已提交索引、提案内容持久化到一个 WAL（Write Ahead Log）日志文件中，用于保证集群的一致性、可恢复性

<figure><img src="../../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

## Leader 选举

### 节点状态

当 etcd server 收到 client 发起的 put hello 写请求后，KV 模块会向 Raft 模块提交一个 put 提案，我们知道只有集群 Leader 才能处理写提案，如果此时集群中无 Leader， 整个请求就会超时。

在raft中，任何节点都处于以下三种状态之一：

Follower，跟随者， 同步从 Leader 收到的日志，etcd 启动的时候默认为此状态；

Candidate，竞选者，可以发起 Leader 选举；

Leader，集群领导者， 唯一性，拥有同步日志的特权，需定时广播心跳给 Follower 节点，以维持领导者身份。

<figure><img src="../../.gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>



以下分析记录etcd  release-3.5

### 节点初始化

<pre class="language-go"><code class="lang-go"><strong>// raft/raft.go 318行
</strong><strong>func newRaft(c *Config) *raft {
</strong>   if err := c.validate(); err != nil {
      panic(err.Error())
   }
   raftlog := newLogWithSize(c.Storage, c.Logger, c.MaxCommittedSizePerReady)
   hs, cs, err := c.Storage.InitialState()
   if err != nil {
      panic(err) // TODO(bdarnell)
   }

   r := &#x26;raft{
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
   if c.Applied > 0 {
      raftlog.appliedTo(c.Applied)
   }
   r.becomeFollower(r.Term, None)

   var nodesStrs []string
   for _, n := range r.prs.VoterNodes() {
      nodesStrs = append(nodesStrs, fmt.Sprintf("%x", n))
   }

   r.logger.Infof("newRaft %x [peers: [%s], term: %d, commit: %d, applied: %d, lastindex: %d, lastterm: %d]",
      r.id, strings.Join(nodesStrs, ","), r.Term, r.raftLog.committed, r.raftLog.applied, r.raftLog.lastIndex(), r.raftLog.lastTerm())
   return r
}
</code></pre>

根据配置文件，初始化一个raft对象，同步持久化数据

调用 r.becomeFollower(r.Term, None)，即所有节点启动时均为follower状态

<pre class="language-go"><code class="lang-go"><strong>// raft/raft.go 680行
</strong><strong>func (r *raft) becomeFollower(term uint64, lead uint64) {
</strong>   r.step = stepFollower
   r.reset(term)
   r.tick = r.tickElection
   r.lead = lead
   r.state = StateFollower
   r.logger.Infof("%x became follower at term %d", r.id, r.Term)
}
</code></pre>

设置raft节点状态及超时选举方法func

### 超时后开启选举

```go
// raft/raft.go 644行
// tickElection is run by followers and candidates after r.electionTimeout.
func (r *raft) tickElection() {
   // 超时计数+1
   r.electionElapsed++
   
   // 判断节点是否可以竞选并且已经超时，则开始竞选
   if r.promotable() && r.pastElectionTimeout() {
      r.electionElapsed = 0
      r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
   }
}

// promotable indicates whether state machine can be promoted to leader,
// which is true when its own id is in progress list.
func (r *raft) promotable() bool {
	pr := r.prs.Progress[r.id]
	return pr != nil && !pr.IsLearner && !r.raftLog.hasPendingSnapshot()
}

// pastElectionTimeout returns true iff r.electionElapsed is greater
// than or equal to the randomized election timeout in
// [electiontimeout, 2 * electiontimeout - 1].
func (r *raft) pastElectionTimeout() bool {
      // 在固定的超时时间上增加一个随机值，避免出现所有节点同时超时的情况
	return r.electionElapsed >= r.randomizedElectionTimeout
}
```

`tickElection`主要是判断是否能够开始选举Leader

### 选举处理逻辑 <a href="#3-xuan-ju-chu-li-luo-ji" id="3-xuan-ju-chu-li-luo-ji"></a>

```go
func (r *raft) Step(m pb.Message) error {
   .....
   switch m.Type {
   case pb.MsgHup:
      // 如果开启预选机制，则开始预选
      if r.preVote {
         r.hup(campaignPreElection)
      } else {
      // 否则直接开始选举
         r.hup(campaignElection)
      }
   .....
}
```

预选机制 --- Follower 在转换成 Candidate 状态前，先进入 PreCandidate 状态，不自增任期号， 发起预投票。若获得集群多数节点认可，确定有概率成为 Leader 才能进入 Candidate 状态，发起选举流程。

<figure><img src="../../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

```go
// raft/raft.go 754行
func (r *raft) hup(t CampaignType) {
   // 是否已经是leader
   if r.state == StateLeader {
      r.logger.Debugf("%x ignoring MsgHup because already leader", r.id)
      return
   }
   // 是否满足竞选条件
   if !r.promotable() {
      r.logger.Warningf("%x is unpromotable and can not campaign", r.id)
      return
   }
   // 是否存在未应用的配置
   ents, err := r.raftLog.slice(r.raftLog.applied+1, r.raftLog.committed+1, noLimit)
   if err != nil {
      r.logger.Panicf("unexpected error getting unapplied entries (%v)", err)
   }
   if n := numOfPendingConf(ents); n != 0 && r.raftLog.committed > r.raftLog.applied {
      r.logger.Warningf("%x cannot campaign at term %d since there are still %d pending configuration changes to apply", r.id, r.Term, n)
      return
   }

   r.logger.Infof("%x is starting a new election at term %d", r.id, r.Term)
   // 正式开始选举
   r.campaign(t)
}
```

```go
// raft/raft.go 777行
// campaign transitions the raft instance to candidate state. This must only be
// called after verifying that this is a legitimate transition.
func (r *raft) campaign(t CampaignType) {
   // 再次检查是否满足竞选条件
   if !r.promotable() {
      // This path should not be hit (callers are supposed to check), but
      // better safe than sorry.
      r.logger.Warningf("%x is unpromotable; campaign() should have been called", r.id)
   }
   var term uint64
   var voteMsg pb.MessageType
   // 根据是否启用预选分别处理
   if t == campaignPreElection {
      // 切换为PreCandidate状态
      r.becomePreCandidate()
      voteMsg = pb.MsgPreVote
      // PreVote RPCs are sent for the next term before we've incremented r.Term.
      term = r.Term + 1
   } else {
   // 切换为Candidate状态
      r.becomeCandidate()
      voteMsg = pb.MsgVote
      term = r.Term
   }
   if _, _, res := r.poll(r.id, voteRespMsgType(voteMsg), true); res == quorum.VoteWon {
      // We won the election after voting for ourselves (which must mean that
      // this is a single-node cluster). Advance to the next state.
      if t == campaignPreElection {
         r.campaign(campaignElection)
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
   // 向所有节点发送消息
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
      r.send(pb.Message{Term: term, To: id, Type: voteMsg, Index: r.raftLog.lastIndex(), LogTerm: r.raftLog.lastTerm(), Context: ctx})
   }
}
```

* 切换到 Candidate 状态
* 发送投票消息给其他节点

### 预选逻辑 <a href="#4-yu-xuan-luo-ji" id="4-yu-xuan-luo-ji"></a>

```go
// raft/raft.go 702
func (r *raft) becomePreCandidate() {
   if r.state == StateLeader {
      panic("invalid transition [leader -> pre-candidate]")
   }
   // Becoming a pre-candidate changes our step functions and state,
   // but doesn't change anything else. In particular it does not increase
   // r.Term or change r.Vote.
   r.step = stepCandidate
   r.prs.ResetVotes()
   r.tick = r.tickElection
   r.lead = None
   r.state = StatePreCandidate
   r.logger.Infof("%x became pre-candidate at term %d", r.id, r.Term)
}

// raft/raft.go 689
func (r *raft) becomeCandidate() {
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

follower 切换成为 candidate 调用了 `r.reset(r.Term + 1)`，把 term +1 了。

而在预选中切换成 PreCandidate 则没有，只是在发送消息的时候，PreCandidate 把消息中的 term +1 而已(没有修改自身状态，只修改消息的Term任期，发起预选)

通过将各个节点**超时时间随机化**，来避免同时开启选举，然后瓜分选票，最终一直无法选出 Leader 的情况。

### 投票结果处理 <a href="#5-tou-piao-jie-guo-chu-li" id="5-tou-piao-jie-guo-chu-li"></a>

```go
// raft/raft.go 1370
// stepCandidate is shared by StateCandidate and StatePreCandidate; the difference is
// whether they respond to MsgVoteResp or MsgPreVoteResp.
func stepCandidate(r *raft, m pb.Message) error {
   // Only handle vote responses corresponding to our candidacy (while in
   // StateCandidate, we may get stale MsgPreVoteResp messages in this term from
   // our pre-candidate state).
   var myVoteRespType pb.MessageType
   if r.state == StatePreCandidate {
      myVoteRespType = pb.MsgPreVoteResp
   } else {
      myVoteRespType = pb.MsgVoteResp
   }
   switch m.Type {
   case myVoteRespType:
      gr, rj, res := r.poll(m.From, m.Type, !m.Reject)
      r.logger.Infof("%x has received %d %s votes and %d vote rejections", r.id, gr, m.Type, rj)
      switch res {
      // 获得了过半选票
      case quorum.VoteWon:
         // 如果预选则发起正式选举
         if r.state == StatePreCandidate {
            r.campaign(campaignElection)
         } else {
         // 如果是正选则竞选成功，转换为leader
            r.becomeLeader()
         // 发送广播给所有节点
            r.bcastAppend()
         }
      case quorum.VoteLost:
         // 如果失败了，不管是预选还是正选都切换成 follower
         // pb.MsgPreVoteResp contains future term of pre-candidate
         // m.Term > r.Term; reuse r.Term
         r.becomeFollower(r.Term, None)
      }
   }
   return nil
}
```

### Leader 心跳 <a href="#6leader-xin-tiao" id="6leader-xin-tiao"></a>

Follower 检测到心跳超时后就会开始选举 Leader，Leader 自然需要不断的给 Follower 发送心跳以保证自己的 Leader 地位。

<pre class="language-go"><code class="lang-go"><strong>// raft/raft.go 718
</strong><strong>func (r *raft) becomeLeader() {
</strong>   //...
   // 设置心跳tick
   r.tick = r.tickHeartbeat
   //...
}
</code></pre>

```go
// raft/raft.go 655
// tickHeartbeat is run by leaders to send a MsgBeat after r.heartbeatTimeout.
func (r *raft) tickHeartbeat() {
   r.heartbeatElapsed++
   r.electionElapsed++

   if r.electionElapsed >= r.electionTimeout {
      r.electionElapsed = 0
      if r.checkQuorum {
         r.Step(pb.Message{From: r.id, Type: pb.MsgCheckQuorum})
      }
      // If current leader cannot transfer leadership in electionTimeout, it becomes leader again.
      if r.state == StateLeader && r.leadTransferee != None {
         r.abortLeaderTransfer()
      }
   }

   if r.state != StateLeader {
      return
   }
   // 超过心跳时间则给follower 发心跳信息
   if r.heartbeatElapsed >= r.heartbeatTimeout {
      r.heartbeatElapsed = 0
      r.Step(pb.Message{From: r.id, Type: pb.MsgBeat})
   }
}
```

调用节点的step方法

```go
func (r *raft) Step(m pb.Message) error {

   switch m.Type {
   default:
      err := r.step(r, m)
      if err != nil {
         return err
      }
   }
   return nil
}
```

Leader的step方法

匹配到心跳类型信息，广播发送心跳

```go
// raft/raft.go 987
func stepLeader(r *raft, m pb.Message) error {
   // These message types do not require any progress for m.From.
   switch m.Type {
   // 心跳消息，广播发送心跳
   case pb.MsgBeat:
      r.bcastHeartbeat()
      return nil
}
```

<pre class="language-go"><code class="lang-go"><strong>// raft/raft.go 524
</strong><strong>// bcastHeartbeat sends RPC, without entries to all the peers.
</strong>func (r *raft) bcastHeartbeat() {
   lastCtx := r.readOnly.lastPendingRequestCtx()
   if len(lastCtx) == 0 {
      r.bcastHeartbeatWithCtx(nil)
   } else {
      r.bcastHeartbeatWithCtx([]byte(lastCtx))
   }
}
</code></pre>

follower 收到心跳信息，重置 electionElapsed 然后再回复一个心跳响应消息。

根据前面 Follower 的逻辑中，每次调用 tick 时，electionElapsed 会+1，如果超过阈值就会发起选举。

然后 Leader 心跳消息时会直接将 electionElapsed 重置，所以如果 Leader 正常运行，Follower 永远不会触发选举。

```go
// raft/raft.go 1415
func stepFollower(r *raft, m pb.Message) error {
   switch m.Type {
   case pb.MsgHeartbeat:
      r.electionElapsed = 0
      r.lead = m.From
      r.handleHeartbeat(m)
}
// raft/raft.go 1513行
func (r *raft) handleHeartbeat(m pb.Message) {
	r.raftLog.commitTo(m.Commit)
	r.send(pb.Message{To: m.From, Type: pb.MsgHeartbeatResp, Context: m.Context})
}
```
