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
