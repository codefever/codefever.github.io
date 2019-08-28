---
title: Phxpaxos源码简略分析
tags:
  - paxos
  - phxpaxos
  - 源码分析
  - 分布式系统
---

> [Phxpoxos](https://github.com/Tencent/phxpaxos)是微信团队在几年前开源的Paxos实现。本文着重从算法实现与代码设计的角度对其进行分析……

## 算法基础

参考资料
* [Paxos made simple](https://www.microsoft.com/en-us/research/publication/paxos-made-simple/)
* [微信 PaxosStore：深入浅出 Paxos 算法协议](https://www.infoq.cn/article/wechat-paxosstore-paxos-algorithm-protocol/)

概念
* Group：Paxos算法的执行机构，包含若干个执行成员。每个Group在数据与逻辑上都是各自独立的。
* Node：这套算法实现的逻辑节点，可以参与多个Group，作为这些Group的执行成员存在。

### BasicPaxos
BasicPaxos通过Acceptor达成的多数派来决定一个value是否被选中。但因为Acceptor是个在分布式环境，多机达成一致是比较困难的，所以算法推导的时候将一部分的决策工作转移给Proposer来做：如果Prepare返回的多数派Acceptor有接收过任意value，Proposer则会将当前Proposal的value更改为其中Proposal Number最新的value，并以当前的Proposal Number广播给Acceptor来接收。

如何确定一个value是否被chosen？个人认为，只有当value所代表的提议被多数派Acceptor接受，且它们Proposal Number大于其他少数派的Proposal Number，则可以确定这个value是被选中的状态。以下用```(PromisedNumber, <AcceptedNumber, value>)```来表示Acceptor的状态，举例子来梳理一把。

    A(1, <1,'hi'>) -- B(4, <2,'hello'>) -- C(3, <3,'world'>) -- D(4, <4,'hello'>) -- E(4, <4,'hello'>)

这个例子里的状态可以通过这样的步骤生成：
1. Proposer①发起```value='hi'```的**提议1**，只得知ABC同意；由于ABC没记录过值，所以可以按提议内容提交value；但最终只写入了A。

        A(1, <1,'hi'>) -- B(1, <>) -- C(1, <>) -- D(0, <>) -- E(0, <>)

2. 某个Proposer②发起```value='hello'```的**提议2**，只得知BCD同意；由于BCD没记录过值，所以可以按提议内容提交value；但最终只写入了B。

        A(1, <1,'hi'>) -- B(2, <2,'hello'>) -- C(2, <>) -- D(2, <>) -- E(0, <>)

3. 某个Proposer③发起```value='world'```的**提议3**，只得知CDE同意；由于CDE没记录过值，所以可以按提议内容提交value；但最终只写入了C。

        A(1, <1,'hi'>) -- B(2, <2,'hello'>) -- C(3, <3,'world'>) -- D(3, <>) -- E(3, <>)

4. 某个Proposer④发起```value='$$$$'```(内容已经不重要了）的**提议4**，只得知BDE同意；其中B曾经接收过提议<2,'hello'>，故将此记录返回给Proposer；Proposer只好将自己提议内容改为```<4,'hello'>```，但只写入了DE。

这个例子里面似乎网络环境非常差，但最后```value='hello'```已经被多数派Acceptor持有了，这是不是就意味着已经被**最终确定**呢？我们可以再发起一轮任意值的Proposal，再来推演一把。

如果这次响应的Acceptor包含DE两者或其中之一，那么Proposal的提议内容会被迫替换成“hello”，```value='hello'```的提议有机会在更多的Acceptor里扩散开来。但如果这次响应的Acceptor只有ABC呢？由于C的AcceptedNumber最大，提议内容会被替换成“world”，那么反倒会很可能导致ABC都接受“world”为最终决议。如下图所示，AB收到了新提议```<5,'world'>```，C保持不变，此时```value='world'```已经成了多数派，且AcceptedNumber大于少数派接受过的提议。

    A(5, <5,'world'>) -- B(5, <5,'world'>) -- C(5, <3,'world'>) -- D(4, <4,'hello'>) -- E(4, <4,'hello'>)

按这样推演下去，因为集合中的两个多数派至少会存在一个交集节点，后续Proposer在Prepare请求中必然会看到这个value以较大的Proposal Number出现，只要网络不完全崩坏，所有Acceptor最终会接受“world”这个结论。

### MultiPaxos
MultiPaxos将BasicPaxos推广到多个value的情况下，只要Prepare通过了，依照Accept的规则可以用相同的Proposal Number给多个value进行提交，有利于在工程实现的时候将value看成是连续单调的log，在此基础上实现[state machine replication](https://en.wikipedia.org/wiki/State_machine_replication)的业务逻辑。

## 接口
* Breakpoint：框架事件回调，用于调试、统计或者打log
* MsgTransport：向Group广播，或给Group中某个节点发包；处理其他节点发来的包，回调给上层
* LogStorage：Paxos日志存储接口
* StateMachine：业务逻辑状态机接口，Execute需要保证有deterministic的结果
* Node：Paxos节点接口，每个Node可同时加入多个Paxos Group，一个Group相当于一个独立的Paxos决策单元。真正实现是在PNode。

## 相对重要的文件或模块

    -- algorithm/   Paxos算法实现
        |- instance
        |- ioloop
        |- committer
        |- acceptor
        |- learner
        |- proposer
    -- communicate/ 网络通信相关
        |- tcp/
        |- udp
        |- dfnetwork
        |- communicate
    -- config/      配置
    -- logstorage/  Paxos日志存储
        |- db
        |- log_store
    -- master/      Leader选主
        |- master_sm
    -- node/        Paxos Node
        |- group
        |- pnode
    -- sm-base/     状态机统一处理
        |- sm_fac

## 源码详解

### communicate
EventLoop、EventBase是比较常见的event-loop实现，基于epoll。

TcpRead、TcpWrite在EventLoop的基础上实现上将tcp的读写分别拆解到多个EventLoop工作线程上，大概是想通过实例化的多个EventLoop来提高事件响应的性能，不过这样建立的TCP链接基本就是当作单工来用的了。另外，UDP也有类似的实现。

DFNetwork是Network的实现，解决针对Group收发包的问题，单点发送到给定组内的某个Node，或者广播到给定组内的所有Node。

Communicate基于Network实现的MsgTransport，并通过Config解决GroupIdx到IP-PORT的映射问题。

总之，通过Communicate和DFNetwork就将底层的收发包问题解决了。

### config
一般配置，如NodeId、GroupId之类。此外，框架内部的系统配置通过StateMachine的形式来管理（比如SystemVSM），Config中也可以获得这部分的信息。

### logstorage
db文件中比较重要的是Database类。Database为日志数据存储的实现，包含了Instance（即paxos日志）、SystemVariables等信息。这里估计是考虑了大value情况下对leveldb这种LSM结构的存储引擎不友好，在实现上将value从leveldb分离出来，单独存储在LogStore中，leveldb中仅保存对该value的索引（FileID+Offset+Checksum，先写LogStore再写leveldb），从而控制了LSM的存储规模，理念上跟pingcap的[titan](https://zhuanlan.zhihu.com/p/55521489)应该差不多。

LogStore类似于数据库设计中的heap file，通过在索引文件之外、在磁盘文件中顺序写入value来提高写吞吐量。一般来说leveldb和LogStore同时写成功的记录才是正确的记录，因此重启时会从db中取出最新一条Instance在LogStore的位置，然后读取后续的heap file尝试修复后续的Instance相关索引。但似乎没有清理策略？

MultiDatabase在此基础上给每个Group维护单独的一个Database。

另外，SystemVariables有必要单独在Storage里做持久化吗，理论上应该是checkpoint的一部分？

### sm-base、StateMachine
SMFac把多个sm的行为看做是单个sm来处理，可参考设计模型中的Compose模式。

另外，StateMachine这块需要注意的一点是，从锁定、释放Checkpoint的methods来看，似乎作者在设计上是希望状态机自身会自动产生Checkpoint，框架上并没有约束状态机在Checkpoint这方面的行为。

### node
PNode是Node接口的实现，自定义服务里需要实例化、初始化PNode，即可在此基础上运行Paxos算法、驱动状态机完成既定的业务逻辑。Group是Paxos组，每个Group的Proposal在各自的组内完成决议，一般会将业务上的数据水平切分给不同的Group，从而通过并行的方式来提高整个系统执行Paxos的决策速度。

这块可以看到一个Proposal的流程是怎么走的，Proposal最终给到指定Group里的Committer来执行。Committer执行的时候会限制同时只能有一个Proposal请求被提出和处理，如果有多个请求同时到达Node，则会在Committer内部变相被排队，直到之前的Proposal执行结束为止。Committer被阻塞超时了、提交冲突了，反正Propose失败了，怎么办？Committer内部先通过重试来解决。

### master
这块应该是充当MultiPaxos中Leader的角色，非常巧妙的是Leader选举是在StateMachine的框架上实现的。这样当定好Leader之后后续Propose请求都直接给到Leader，就不会因为频繁Propose引起组提交性能的波动，且可以在Leader不变的情况下只做Accept。

不过对Leader状态的使用在内部并没有体现，难道是需要外部调用Propose前自行做决策？

### algorithm
首先说一下IOLoop，Paxos Group的执行体依赖的正是IOLoop线程，无论是发起Proposal、超时控制还是收发包，最终都会落到IOLoop线程来统一处理，犹如“消息总线”般的存在。这样可以一定程度避免多线程并发访问的问题，同时Paxos算法也更适合在线性逻辑下实现。

算法实现按角色拆分到不同的类来实现，有Committer、Proposer、Acceptor、Learner等，构成Paxos Instance（这个名字可能会有些混淆，不是Log Entry的概念）。我们可以按一个Proposal流程来看下这些类是怎么交互的。

1. 首先是Committer向IOLoop提交Proposal，记录当前的上下文信息，并阻塞其他并发的Proposal进入。
2. Proposer拿到Proposal，向各个节点发起Prepare（到3）。（*REQ)
3. Acceptor拿到Prepare请求，执行Prepare逻辑，给Proposer回包。（参考算法的第一阶段）
4. Proposer根据Prepare结果，决定是否发起Accept。（*REQ)
5. Acceptor拿到Accept请求，执行Accept逻辑，给Proposer回包。（参考算法的第二阶段）
6. Proposer根据Accept结果，决定是否可以提交记录，可以的话会发起ProposerSendSucc的请求。（*REQ）
7. Learner收到ProposerSendSucc，提交最近的Proposal到状态机。此时如果是本地发起的状态，则可以从Committer找到上下文信息，给PNode那边返回执行结果。
8. Learner学成后，各个角色的InstanceID递进1，可以开始下一轮Proposal了……这个过程中，数据包都是在IOLoop通过Instance流转到各个角色来完成动作的。

从上可以发现，最惨烈的情况下提交一个Proposal需要近3次收发包。这里会有一些优化，我能看出来的主要有两点：

* 如果Proposer成功发起过Prepare，则可以跳过Prepare阶段直接开始Accept（[proposer.cpp:181](https://github.com/Tencent/phxpaxos/blob/master/src/algorithm/proposer.cpp#L181)）。
* 另外，在开启跳过Prepare、连续Accept的情况下，如果Acceptor收到的Accept请求的InstanceID是当前ID+1、且发起方的ProposalNumber不变，则会直接转发当前已接受的Instance给Learner学习（[instance.cpp:611](https://github.com/Tencent/phxpaxos/blob/master/src/algorithm/instance.cpp#L611)），然后等到Learner学成，Acceptor恰好又可以接受当前的请求（[instance.cpp:621](https://github.com/Tencent/phxpaxos/blob/master/src/algorithm/instance.cpp#L621)）。

至此几乎变成单次收发包就可以提交Proposal了。

MultiPaxos里并没有给出Instance（这里是指Log Entry）chosen状态怎么传递的解决方案。我理解phxpaxos在这方面的解法是，将Instance的决议作为一个串行、阻塞的过程来执行，需要Proposal顺利从Accept走到Learn达到chosen状态，才能执行下一个Instance的决议，否则会停留在最后一个Instance上重试，通过类似BasicPaxos的方法要么找回已被chosen的Instance，要么重新选定一个Instance。所以在服务启动时，PlayLog会执行最后一个Instance之前的记录，最后一个Instance需要在运行时重新进行一次决议才能最终敲定下来是否chosen。相比于Raft里的CommitIndex，通过记录最后一条chosen的Instance位置来表明在此及之前的Instance都已被chosen，这种方式似乎在简单之余更适合Paxos，毕竟其原生没有Log Match的方法没法直接套用CommitIndex，这样不失为一种折中办法。

## 总结
phxpaxos采用了一些自己的手段实现了MultiPaxos，确保了Paxos日志的连续性，从而也确保了statemachine replication在多机环境下的执行正确性。

个人认为，Paxos相比Raft的优势是Instance是可以有空洞的，如果需要的数据只与最后chosen的Instance有关，则用Paxos会非常高效，比如微信的PaxosStore就是个例子。但如果是状态机模式需要以日志连续性为基础，我觉得还是用Raft来实现会更舒服些，至少有严格的论证和明确的实现方法。

Lamport在MultiPaxos上实在留白太多了！
