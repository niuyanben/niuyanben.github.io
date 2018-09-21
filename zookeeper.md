## ZooKeeper 的功能和原理

ZooKeeper 是一个开源的分布式协调服务，由雅虎创建，是 Google Chubby 的开源实现。
分布式应用程序可以基于 ZooKeeper 实现诸如数据发布/订阅、负载均衡、命名服务、分布式协
调/通知、集群管理、Master 选举、配置维护，名字服务、分布式同步、分布式锁和分布式队列
等功能。

### ZooKeeper基本概念

#### 集群角色

夺Zookeeper中，有三种角色
1. Leader
2. Follower
3. Observer

一个 ZooKeeper 集群同一时刻只会有一个 Leader，其他都是 Follower 或 Observer。

ZooKeeper 配置很简单，每个节点的配置文件(zoo.cfg)都是一样的，只有 myid 文件不一样。myid 的值必须是 zoo.cfg中server.{数值} 的{数值}部分。

zoo.cfg配置文件示例 
```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
clientPort=2181
server.1=IP1:2888:3888
server.2=IP2:2888:3888
server.3=IP3:2888:3888
```

在装有 ZooKeeper 的机器的终端执行 `zookeeper-server status` 可以看当前节点的ZooKeeper是什么角色（Leader or Follower）。

ZooKeeper 默认只有 Leader 和 Follower 两种角色，没有 Observer 角色。为了使用 Observer 模式，在任何想变成Observer的节点的配置文件中加入`:peerType=observer` 并在所有 server 的配置文件中，配置成 observer 模式的 server 的那行配置追加 `:observer`

#### 节点读写服务分工
1. ZooKeeper 集群的所有机器通过一个 Leader 选举过程来选定一台被称为『Leader』的机器，Leader服务器为客户端提供读和写服务。

2. Follower 和 Observer 都能提供读服务，不能提供写服务。两者唯一的区别在于，Observer机器不参与 Leader 选举过程，也不参与写操作的『过半写成功』策略，因此 Observer 可以在不影响写性能的情况下提升集群的读性能。

#### Session
Session 是指客户端会话，在讲解客户端会话之前，我们先来了解下客户端连接。在ZooKeeper 中，一个客户端连接是指客户端和 ZooKeeper 服务器之间的TCP长连接。

ZooKeeper 对外的服务端口默认是2181，客户端启动时，首先会与服务器建立一个TCP连接，从第一次连接建立开始，客户端会话的生命周期也开始了，通过这个连接，客户端能够通过心跳检测和服务器保持有效的会话，也能够向 ZooKeeper 服务器发送请求并接受响应，同时还能通过该连接接收来自服务器的 Watch 事件通知。

Session 的 SessionTimeout 值用来设置一个客户端会话的超时时间。当由于服务器压力太大、网络故障或是客户端主动断开连接等各种原因导致客户端连接断开时，只要在SessionTimeout 规定的时间内能够重新连接上集群中任意一台服务器，那么之前创建的会话仍然有效。


#### 数据节点
zookeeper的结构其实就是一个树形结构，leader就相当于其中的根结点，其它节点就相当于follow节点，每个节点都保留自己的内容。

zookeeper的节点分两类：持久节点和临时节点
1. 持久节点（PERSISTENT） 
所谓持久节点，是指在节点创建后，就一直存在，直到有删除操作来主动清除这个节点——不会因为创建该节点的客户端会话失效而消失。
2. 持久顺序节点（PERSISTENT_SEQUENTIAL）  
这类节点的基本特性和上面的节点类型是一致的。额外的特性是，在ZK中，每个父节点会为他的第一级子节点维护一份时序，会记录每个子节点创建的先后顺序。基于这个特性，在创建子节点的时候，可以设置这个属性，那么在创建节点过程中，ZK会自动为给定节点名加上一个数字后缀，作为新的节点名。这个数字后缀的范围是整型的最大值。 
在创建节点的时候只需要传入节点 “/test_”，这样之后，zookeeper自动会给”test_”后面补充数字。
3. 临时节点（EPHEMERAL） 
和持久节点不同的是，临时节点的生命周期和客户端会话绑定。也就是说，如果客户端会话失效，那么这个节点就会自动被清除掉。注意，这里提到的是会话失效，而非连接断开。另外，在临时节点下面不能创建子节点。 
这里还要注意一件事，就是当你客户端会话失效后，所产生的节点也不是一下子就消失了，也要过一段时间，大概是10秒以内，可以试一下，本机操作生成节点，在服务器端用命令来查看当前的节点数目，你会发现客户端已经stop，但是产生的节点还在。
4. 临时顺序节点（EPHEMERAL_SEQUENTIAL） 
此节点是属于临时节点，不过带有顺序，客户端会话结束节点就消失。

下面是一个利用该特性的分布式锁的案例流程。 
1. 客户端调用create()方法创建名为“locknode/ 
guid-lock-”的节点，需要注意的是，这里节点的创建类型需要设置为EPHEMERAL_SEQUENTIAL。 
2. 客户端调用getChildren(“locknode”)方法来获取所有已经创建的子节点，注意，这里不注册任何Watcher。 
4. 如果在步骤3中发现自己并非所有子节点中最小的，说明自己还没有获取到锁。此时客户端需要找到比自己小的那个节点，然后对其调用exist()方法，同时注册事件监听。 
5. 之后当这个被关注的节点被移除了，客户端会收到相应的通知。这个时候客户端需要再次调用getChildren(“locknode”)方法来获取所有已经创建的子节点，确保自己确实是最小的节点了，然后进入步骤3。


#### 状态信息
每个 节点除了存储数据内容之外，还存储了 节点本身的一些状态信息。用 get 命令可以同时获得某个 节点的内容和状态信息

在 ZooKeeper 中，version 属性是用来实现乐观锁机制中的『写入校验』的（保证分布式数据原子性操作）。

#### 事物操作
在ZooKeeper中，能改变ZooKeeper服务器状态的操作称为事务操作。一般包括数据节点创建与删除、数据内容更新和客户端会话创建与失效等操作。对应每一个事务请求，ZooKeeper都会为其分配一个全局唯一的事务ID，用 ZXID 表示，通常是一个64位的数字。每一个 ZXID对应一次更新操作，从这些 ZXID 中可以间接地识别出 ZooKeeper 处理这些事务操作请求的全局顺序。

#### Watcher(事件监听器)
Watcher是 ZooKeeper 中一个很重要的特性。ZooKeeper允许用户在指定节点上注册一些 Watcher，并且在一些特定事件触发的时候，ZooKeeper 服务端会将事件通知到感兴趣的客户端上去。该机制是 ZooKeeper 实现分布式协调服务的重要特性。
