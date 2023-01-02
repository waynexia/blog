---
title: "Paper Reading: CURP protocol"
date: 2022-09-18 17:58:33
tags: "Paper Reading"
mathjax: true
---
论文名字是《Exploiting Commutativity For Practical Fast Replication》

CURP全称Consistent Unordered Replication Protocol，主要是利用操作之间的相关性（Commutative）来实现局部乱序提交的协议。这里Commutative指的是不同操作记录之间是否可以交换执行顺序而不影响最终结果，比如`x=1`、`y=2`和`x=3`三条记录中`y=2`与其他两条都是无关的，满足Commutative可以乱序执行，而`x=1`与`x=3`之间有关系，需要一个确定的执行顺序。

相比于其他的复制协议，CURP的特点是对于满足Commutative的记录可以以1 RTT的延时完成提交。

# 集群角色
除了常见的`Client`，`Leader`，`Backup`之外CRUP中还引入了一个新的`Witness`角色。简单概括一下这些角色：
- `Client`
    - 与`Witness`和`Leader`交互，对`Witness`的通信是广播的形式(同时与所有`Witness`通信)  
    - 自己也有一定的状态，主要是关于操作的执行情况的。
- `Witness`  
	- CURP协议定义的角色，存在复数个实例，且实例之间对等  
	- 有存储能力来临时存储近期的操作记录，但不需要维护状态机（操作结果），就是一个WAL
    - 需要感知具体的操作内容并判断操作之间是否有可交换性(commutative)  
	- 实例间独立运作及决策，也独立于`Leader`/`Backup`。
- `Leader`
	- 集群主节点，有其他主备协议中常见的`Leader`的能力(维护状态机，答复读写请求，数据同步等)  
	- 同样需要感知具体操作并判断可交换性
	- 可以在把操作同步给`Backup`之前答复请求
    - 论文中叫`Master`
- `Backup`
    - 普通的backup角色

在论文的正文部分中`Witness`都是作为一个独立的部署实例来讨论的，不过附录里也提到实践中也没必要将`Witness`单独部署，而是跟着`Leader`/`Backup`一起运作，只作为逻辑上的一个独立单元。所以一些RPC请求实际上是可以合并（比如`Client`同时请求`Leader`和与其在一起的`Witness`）或者是本地IPC（比如`Leader`/`Backup`请求对应的`Witness`）的。因此对于实际使用的时候来说`Witness`更像是CURP引入的新机制的一个概括抽象。

# 基本操作
协议的具体内容可以从几个基本操作来说明

## 更新
由`Client`发起，`Client`同时对`Leader`及**所有**`Witnesses`发出请求，根据这个请求与之前的请求是否无关(可交换，commutative)有两种情况：  
- 如果无关，那么`Leader`和**所有**`Witnesses`都会accept这个请求。`Client`在收到**全部**的accept答复之后可以当这个请求已完成。
- 如果`Leader`或任意一个 `Witness` 认为有关(not commutative)，`Client`都需要要求`Leader`进行一次sync，将状态与`Backups`进行同步后答复`Client`。`Client`可以依据这个sync请求的答复将请求完成，这种情况下`Witness`的答复没有影响。

也就是说，一次操作从`Client`发起到完成一共有三个状态：`Proposed`，`Unsynced`和`Synced`。`Client`会对除了`Backup`之外的所有节点发起请求，在上面的第一种情况下成为`Unsynced`，此时`Client`已经可以认为该操作完成，`Leader`也可以把操作应用到状态机上。`Leader`应用`Unsynced`操作的过程叫做“推测执行（speculative execution）”。`Unsynced`的操作会在一下次收到sync请求时往`Backups`进行同步，同步完成的操作称为`Synced`。而在上面的第二种情况下操作会跳过`Unsynced`这一步。

但是从`Leader`视角看一个操作只要`Leader`认为是commutative的它就会应用到状态机，但这个操作可能会被`Witness`拒绝（认为是not commutative）。这时候需要依赖`Client`来发起sync请求，要求`Leader`与`Backups`进行同步。

## Leader 故障恢复
新 `Leader` 的产生/恢复有两步，首先同步 `Backup` 的状态，然后随便挑一个 `Witness` 来重放没同步到 `Backup` 中的操作。

同步`Backup`状态就获得了所有`Synced`的操作，而从上面的更新流程可以知道`Unsynced`的操作存在于**全部**的`Witnesses`中，所以这里随便挑一个 `Witness` 都有全部所需的信息。另外`Unsynced`操作在不同的`Witnesses`上所提交的顺序可能不一样，所以CURP要求只有满足commutative可以乱序执行的操作才可以推测执行。

另外还需要解决`Backup`和`Witness`可能存在重复记录的问题，如果将已经是`Synced`的操作再进行重放是会导致状态错误的。因此CURP需要保证每个操作只能执行一次，简单来说就是让`Client`给每个操作加上一个`RPC ID`。
>To avoid duplicate executions of the requests that are already replicated to backups, CURP relies on exactly-once semantics provided by RIFL, which detects already executed client requests and avoids their re-execution

当然如果是一个 `Backup` 成为新 `Leader` 就可以跳过第一步。

## 读取
`Client` 可以直接从 `Leader` 读，不过需要这次读操作也与其他 `Unsynced` 的操作无关，否则需要先 sync 一下，因为对同一个内容的读和写操作是无法乱序执行的。而在`Leader`视角看，推测执行的操作不一定在重启/变更Leader之后还存在（在`Leader`应用到状态机后，但是被`Witnesses`记录下来之前发生crash），所以读推测执行的内容也需要检查commutative。所以这种读操作与更新操作的区别只有(1)不涉及 `Witness` (2)不会影响 `Leader` 的状态机，因此不需要被记录。

`Client` 也可以从 `Backup` 读，不过读之前需要先向一个 `Witness` 询问这个操作是否 commutative。只有在得到肯定答复的情况下才可以读 `Backup`，否则还是只能从 `Leader` 读。前面要求一个操作被记录到所有 `Witnesses` 也考虑到了这里能够随意请求 `Witness` 来询问 commutative。

同时因为`Backups`之间的状态可能因为同步进度不同而出现差异，为了从任意一个 `Backup` 都能读到同样的结果，`Backups` 之间还需要感知互相的同步状态(即一个操作是否被所有的 `Backups` 同步完成)。
>even if a value is replicated to some of backups, the value may get lost if the master crashes and a new master recovers from a backup that didn’t receive the new value. A simple solution for this problem is that backups don’t allow reading values that are not yet fully replicated to all backups.

## GC
这里讨论`Witness' records`和`RPC ID`两个资源的回收。

从前面可知，`Witness` 只有在新记录与现在所有记录都无关的情况下才能 accept。所以为了提高成功率 `Witness` 必须尽快丢掉不必要的数据(被 `Leader` 同步到了 `Backups` 上，即synced的数据)。在 CURP 中 `Leader` 会给 `Witness` 发 GC 指令，告诉 `Witness` 哪些操作已经完成了同步并可以丢弃。

在 `Witness GC` 和更之前的故障恢复环节都需要对操作记录进行标识，CPUP 使用的是 `RIFL` 中的 `RPC ID` 来完成。因为这个 ID 是 Client 分配的，所以无法使用累进确认的方式而必须对每个 ID 都进行记录。此外 CURP 还对 `RIFL` 做了一些改动，在论文的 *附录§C.1* 节有展开，主要是为了适配 Witness 提供的 Recovery。

# 进阶操作
从更新操作的步骤可以看到 `Client` 向 `Witnesses` 写数据以及 `Leader` 向 `Backups` 同步状态都是有全同步的要求的，即要求所有的请求对象都完成才能算完成，而后面的故障恢复和读取操作也都用到了这一点。

这样操作在网络分区或者单节点处理慢之类情况下的可用性和性能是比较让人担心的。论文在附录还展开了另一种方法，可以让 `Client` 向 `Witnesses` 写数据时只得到*超大多数(superquorum)* `Witnesses` 的答复就行。这里“超大多数”的计算方式是 $$superquorum=f+\lceil f/2 \rceil + 1$$，其中`f`是能容忍多少节点挂掉的数量。这里比传统多数派还多了一个“少数派的大多数”的原因是为了故障恢复时剩下的 `Witnesses` 之间还能保持多数派。
>The reason why CURP needs a superquorum instead of a simple majority is to ensure commutativity of replays from witnesses during recovery

而相应的，`Leader`在恢复的时候以及`Client`在读`Backup`之前查询的时候都需要收集多数派`Witnesses`的结果。其实就是牺牲了一些读时的便利性来优化写的操作。

此外 `Leader` 向 `Backups` 同步数据也是类似，只要同步到多数派上，在恢复的时候选最新的。在 `Client` 读 `Backup` 的时候也选多数派和最新任期，类似 Raft 的那一套操作。

# 一些证明
论文在 *附录§A* 里面简单证明了一下 `Durability`, `Consistency` 和 `Linearizability` 几个关键特性，不过都是基于基本操作的版本（全同步写）。这里也洗一下稿。

先回顾一下几个操作时的规则：
- `Client` 只有两种情况下可以完成一个操作：该操作被**所有** `Witnesses` 记录 *或* 该操作被**所有** `Backups` 持久化(即被 `Leader` 接收)
  >from §3.2.1, a client only completes an update operation if (1) it is recorded in all f witnesses or (2) it is replicated to f backups.
- 一个未同步(unsynced)的操作必须与所有其他未同步的操作无关(commutative)
  >a completed unsynced operation must be individually commutative with all preceding operations that are not synced yet.
- 如果 `Leader` 要答复一个不可交换(not commutative)的操作，需要先与 `Backup` 进行一次同步(sync)
  >a master must sync before responding if the current operation is not commutative with any other existing (preceding) unsynced operations.

## Durability
所有的操作至少会存在于所有 `Backups` (synced) 或所有 `Witnesses` (unsynced) 上，所以恢复的时候找一个 `Backup` 加上一个 `Witness` 就能拿到所有的信息。

## Consistency
synced 的操作存在在 Backup 上，并通过 ID 的方式防止重复执行。unsynced 的操作存在在 Witness 上，并且可以以任意顺序重放。

不过论文好像有一个内容没讲，就是 `Witness` 上记录的操作不一定最终会执行（比如`Client`发起sync之前发生crash），所以 `Witness` 是需要有机制去确认一个记录下来的操作是有效的。我觉得可以让 `Leader` 定期给 `Witness` 同步这个信息（比如将这个也作为 sync 的一步），同时 `Witness` 在恢复期间不提供未被确认的操作。

## Linearizability
还是从前面的规则推导。如果一个写入在只被部分 `Witness` 记录到的时候就crash，重启之后的 `Leader` 可能不会有这个信息，但是这时候 `Client` 也不会认为这个操作已完成，因为没收到所有 `Witness` 的答复。对于`Client`认为已经完成的操作再去读的时候 `Leader` 能保证线性一致。读 `Backup` 的流程论文的图4也稍微涉及到了线性一致的几种情况。

不过这里只是文字论述，也没有什么并法内容，作为证明感觉还是有点单薄，但是也想不到什么反面例子……

# 最后
这篇文章省略了论文中的一些内容，比如一些实现上的经验细节，分区的处理等等。看下来觉得基本思想还好理解，论文说这是一个通用的优化手段，但是怎么变成一个真正的优化感觉还是要在实现的时候多做些微操……