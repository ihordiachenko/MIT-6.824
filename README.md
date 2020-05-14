

# MIT 6.824 

赶在返校的最后十几天完成了[6.824: Distributed Systems](https://pdos.csail.mit.edu/6.824/schedule.html)，最后剩2个Challenge和思路返校之后慢慢补吧。

## Status

- [x] Lab 1
- [x] Lab 2
- [x] Lab 3
- [x] Lab 4
  - [ ] Challenge 1
  - [ ] Challenge 2

## LAB1 MAPREDUCE

第一个实验是完成一个简单的分布式mapreduce框架，整个实验分三步走:

### 1 完成worker

worker的主体是一个for循环，不断的向master 调用**AskTask RPC**。然后执行不同的回调函数:**doMap  doReduce**.

**doMap**负责依次读取master获取的输入文件列表，调用mapf,存取中间结果

**doReduce** 负责读取<u>对应</u>的中间结果文件，调用reducef,存取输出

大致完成后，可以修改mrsequential.go文件，以串行的方式模拟调用worker里的函数，比较下输出是否正确

### 2 完成master

master负责对任务的调度。整个运行过程分两种任务，**MAP** 和 **REDUCRE**

```go
type Task struct {
  taskType_ int
  isCompleted_ bool
  isDistributed_ bool
  index_ int
}
```

分别用两个list来保存。每次收到来自worker的**Asktask** RPC请求，调用 tryGetTask尝试获取一个当前阶段尚未分配的task.在执行不同阶段时，得到的task种类也是不一样的。例如倘若在map阶段，所有的map都已经被分配了,tryGetTask会返回一个类型为NONE的task.worker收到之后，应该暂时sleep几百毫秒之后再次请求.

完成master之后基本可以通过前几个测试.

### 3 完成fault-tolerance

第三部主要是针对最后一个测试。最后一个测试会随机阻塞当前的任务，master超时之后，会重新分配这个任务。但由于前一个worker执行该任务时候，可能只把结果写了一半，所以如果什么都不做的话，最后总的结果大概率会出错。所以这个步骤就是主要解决map和reduce过程中文件的存储问题.

课程已经给出了相应的解决方案，首先创建一个tempfile,然后进行写入。在写入彻底完成后，把这个tempFile rename 成对应的文件名.

当master分配出一个任务之后，应该启动一个定时协程**waitForTask**。worker完成任务之后，应该调用**submitTask** RPC，master将对应的task.isCompleted_设置为true.当**waitForTask**超时之后，会检查对应的tasK的完成情况，没有完成的话应该设置isDistributed__为false,以便重新分配没按时完成的task.



## LAB2 Raft

整个lab2是要求你实现一个精简版Raft，只实现Raft最核心的两个RPC----RequestVote和AppendEntries.对于Raft算法，已经有很多开源了的，例如[braft](https://github.com/baidu/braft) [ectd](https://github.com/etcd-io/etcd) 等。这些都是非常值得学习的算法库，但对初学者却不够友好，因为通常在工业生产中会对Raft做一些细节上的改动，如PreVoted，同时为了性能做了复杂的并发模型并且整个库还包括其他部分如存储系统。所以824的这个lab2是很适合Raft的初学者加深对基础算法的理解。

这里建议在进行动手之前好好看看课程给出的[文档](https://thesquareplanet.com/blog/students-guide-to-raft/)和论文的Figure-2。简单提几个要点：

在RPC通信开始前，保证mutex Unlock,以免整个系统造成死锁。

在处理任何来自其他peer的信息之前，首先应该updateTerm---如果发送方的Term > currentTerm,节点应该step down.

确保Leader不在当前任期之前更新commitIndex.

#### RequestVote

RequestVote整个过程详见Figure-2.

首先更新状态，vote for self。同时应该reset election 计时器。

```go
rf.CurrentTerm++
rf.VotedFor = rf.me
rf.role = CANDIDATE
```

然后发起投票，伪代码如下

```
for i := 0; i < len(rf.peers); i++ {
		if i == rf.me {
			continue
		}
		go func() {
			//sendrequestvote
			ok := rf.sendRequestVote()
			if ok {
				ok = rf.handleRequestVoteResponse()
			}
			finished++
			if ok {
				granted++
			}
		}()
	}
```

这里需要注意一下，当满足以下两种情况，应该立即结束投票：

所有的投票已经全部完成，无论成功与否。

已收到足够的投票。

结束投票之后判断是否可以becomeLeader.

#### RequestVoteHandle

按照Figure-2实现即可。

### AppendEntries

AppendEntries用于发送日志与心跳连接。按照论文所说，在心跳连接时，发送一个或多个LogEntry也是可以接受的。具体过程还是参考Figure-2.但是实现时有一点需要注意，当接受到RPC 返回消息时，应该进一步判断是否需要继续传送，让follower跟上进度。

### roll back quickly

当follower收到来自leader的AppendEntries消息,发现日志不匹配时，会reply false.leader收到之后，会讲nextIndex - 1,再次进行尝试。但当follower落后太多的情况下，这种方法的效率就显得很低。论文中其实给出了一种优化方式，在824的课程笔记里，进一步进行了介绍。 

> ```
> Case 1            Case 2       Case 3
> S1: 4 5 5       4 4 4        4
> S2: 4 6 6 6 or  4 6 6 6  or  4 6 6 6
> rejection from S1 includes:
> XTerm:  term in the conflicting entry (if any)
> XIndex: index of first entry with that term (if any)
> XLen:   log length
> Case 1 (leader doesn't have XTerm):
> 	nextIndex = XIndex
> Case 2 (leader has XTerm):
> 	nextIndex = leader's last entry for XTerm
> Case 3 (follower's log is too short):
> 	nextIndex = XLen
> ```

## LAB3 

Lab3需要用到之前的Raft库，搭建一个Key/value service。整个实验相对来说比较简单，只说几个要点。

### client 与 replicas 的交互

​	要与复制组交互的client首先必须知道replicas的配置信息（configuration），最开始client不知道replicas中哪一个server是leader.所以可以随机向一个server发出应用请求。在Raft论文中给出的响应方式是：

```
if isLeader {
		service()
} else {
	return cachedLeader
}
```

​	在这里我们简单实现成

```
if isLeader {
		service()
} else {
	return ErrorWrongLeader
}
```

如果server 不是 leader ，client会受到ErrorWrongLeader的错误返回，并且继续向其他server发送请求。

### 响应超时

​	在leader进行service时，如果出现network partition现象，leader无法同步日志并且无法意识到自己应该step down,而原先的service则会一直阻塞。因此在一开始尚未进行service时，需要添加一个定时任务，超时之后若还是没完成service,则应该返回 ErrorWrongLeader错误。

### 一致性保证

​	Raft算法本身不提供保证**强一致性**保证，但在论文里给出相应的实现方式。这个我的实现是，所有的server在内存缓存

所有client最后操作的时间戳，每一个client的时间戳在client产生RPC request之时决定，只有当改时间戳严格大于最后时间戳，才能进行操作。否则直接忽略，返回成功。