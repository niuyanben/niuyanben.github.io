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
1. ZooKeeper 集群的所有机器通过一个 Leader 选举过程来选定一台被称为『Leader』
   的机器，Leader服务器为客户端提供读和写服务。

2. Follower 和 Observer 都能提供读服务，不能提供写服务。两者唯一的区别在于，
Observer机器不参与 Leader 选举过程，也不参与写操作的『过半写成功』策略，因
此 Observer 可以在不影响写性能的情况下提升集群的读性能。

#### Session
Session 是指客户端会话，在讲解客户端会话之前，我们先来了解下客户端连接。在
ZooKeeper 中，一个客户端连接是指客户端和 ZooKeeper 服务器之间的TCP长连接。

ZooKeeper 对外的服务端口默认是2181，客户端启动时，首先会与服务器建立一个TCP
连接，从第一次连接建立开始，客户端会话的生命周期也开始了，通过这个连接，客户端能够通
过心跳检测和服务器保持有效的会话，也能够向 ZooKeeper 服务器发送请求并接受响应，同
时还能通过该连接接收来自服务器的 Watch 事件通知。

Session 的 SessionTimeout 值用来设置一个客户端会话的超时时间。当由于服务器

#### 数据节点
zookeeper的结构其实就是一个树形结构，leader就相当于其中的根结点，其它节点就相当于
follow节点，每个节点都保留自己的内容。

zookeeper的节点分两类：持久节点和临时节点
1. 持久节点：
所谓持久节点是指一旦这个 树形结构上被创建了，除非主动进行对树节点的移除操
作，否则这个 节点将一直保存在 ZooKeeper 上。

2. 临时节点：
临时节点的生命周期跟客户端会话绑定，一旦客户端会话失效，那么这个客户端创
建的所有临时节点都会被移除。
