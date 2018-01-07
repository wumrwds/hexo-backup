---
title: Redis主从复制（Replication）的配置及主从切换（Failover）的解决方案
toc: true
date: 2017-08-18 13:20:18
tags: Redis
categories: NoSQL
---

在实际工程中，若使用单台的服务器部署Redis会使单点负载过高，并且存在发生单点故障导致系统崩溃的可能。因此实际工程中一般使用主从复制（Replication）或集群（Cluster）的方式来提高系统的稳定性及容灾能力。

本文将简单介绍Redis主从复制（Replication）的实现机制及配置使用。

<!-- more -->

### 一、主要机制

主从复制简单来说就是使多台Slave Server上拥有与Master Server上相同的数据的精确备份。

Redis的主从复制的实现主要依赖下面的三个机制：

1. 当主机（Master）与从机（Slave）建立连接后，主机将会通过向从机发送命令流（stream of commands）的方式使从机保持数据的更新。从机可以通过获得的命令流来复制主机对数据集的操作，从而完成数据的同步。
2. 当主机与从机之间的连接由于网络原因或监听超时（timeout）而断开时，从机将会尝试重连主机并获取丢失部分的同步命令流进行再同步（resynchronization）。
3. 当部分再同步（partial resynchronization）的方式失败时，从机将会向主机获取一个完全的再同步（full resynchronization）。主机此时将会对其当前的数据集创建一个快照（snapshot），将其发送给从机后，之后按照原先的模式，有改动再向从机发送命令流，从而保持主从同步。

### 二、 主要特性

Redis的主从复制有如下的几个特性：

* 异步（asynchronous）
* 一台主机可以有多个从机
* 一台从机可以接受其他从机的连接，因而级联结构的从机结构也是可行的，并且自Redis 4.0后，所有的从机都能够从主机收到相同的复制命令流（包括级联模式的从机）
* Redis的主从复制在主机端是非阻塞的。这表示主机在处理从机的复制请求时，也能处理正常的查询请求。

- Redis的主从复制在从机端是非阻塞的。默认配置下，从机在执行主从同步的同时，也可以处理查询请求。查询请求将使用旧版本的数据集。当然，也可以通过配置使从机在主从复制时对查询请求返回error。但是，当新数据获取完成时，从机需要删除旧版本数据，载入新版本的数据，而这个过程将是阻塞的。自Redis 4.0可通过配置使删除旧数据的操作运行在另一个线程上，但是主线程上的载入新数据的操作仍将使从机阻塞。
- 可以使用主从复制来免除主机写入完整的数据到磁盘的开销：通过配置主机的redis.conf使主机不进行将数据持久化至磁盘的操作，连接到从机使其通过RDB或AOF的方式保存数据。但是这种方式在使用时需要小心，当主机重启时将会变为空数据集，而此时若是从机尝试与主机同步，两边的数据都将变为空数据集，数据丢失。

### 三、主机配置

下面介绍主从复制的配置。

在机器上下载好Redis之后，解压，将redis.conf复制一份。

首先，建议先将redis.conf中的bind配置为(主、从)机器本身的IP，并设置为后台守护进程模式

> daemonize yes
>
> bind server_ip

之后，主机不需要改动，主要对从机的配置进行修改。

### 四、从机配置

从机之前操作与主机配置相同，修改完以上的配置之后，继续修改redis.conf中的 slaveof 配置

> slaveof \<master_ip\> \<master_port\>

修改好之后，建议启动前先查看是否存在rdb或aof持久化文件，若存在，最好先清除，防止影响同步。

最后，使用时可切入Redis的bin目录下，输入以下命令进行启动。注意，先启动主机再启动从机。

> ./redis-server redis.conf -h \<server\_ip> -p \<redis\_port\>

进入redis-cli输入info命令可查看主从复制的配置是否成功。

### 五、主从切换的配置

完成上面的配置之后，就已经配置好Redis的主从复制功能了，但是当主机crash了，我们的Redis服务并不能进行相应的主从切换，使从机取代主机。

这时，我们可以采用Redis Sentinel解决这一问题。

Redis Sentinel为Redis提供高可用性（HA， High Availablity），具体具有监视（Monitoring）、通知（Notification）、自动切换（Automatic failover）、配置提供（Configuration provider）的功能。并且Redis Sentinel是一个分布式系统，多个Sentinel将监视master和slave是否良好运行，如果发现某个redis节点运行出现状况，Sentinels能够通知另外一个进程(例如它的客户端)；而当一个master节点不可用时，能够选举出master的多个slave(如果有超过一个slave的话)中的一个来作为新的master,其它的slave节点会将它所追随的master的地址改为被提升为master的slave的新地址。

下面本文仅使用单台的Sentinel进行演示。

首先，从解压出的Redis中，复制一份sentinel.conf。作大致如下的修改：

> \#redis-0
>
> \#sentinel实例之间的通讯端口
>
> port 26379
>
> logfile "./sentinel.log"
>
> \#master1
>
> sentinel monitor master1 master\_ip master\_port 1
>
> sentinel down-after-milliseconds master1 5000
>
> sentinel failover-timeout master1 900000
>
> sentinel can-failover master1 yes
>
> sentinel parallel-syncs master1 2
>
>
> \#master2  可以添加多组主从的redis监听
>
> ...

其中 `sentinel monitor master1 <master_ip> <master_port> <quorum>`， 配置中的注释对\<quorum\>进行如下的描述：

> Tells Sentinel to monitor this master, and to consider it in O_DOWN
>
> (Objectively Down) state only if at least \<quorum\> sentinels agree.


`sentinel down-after-milliseconds <master-name> <milliseconds>`， 配置中的注释对\<quorum\>进行如下的描述：

> Number of milliseconds the master (or any attached slave or sentinel) should
>
> be unreachable (as in, not acceptable reply to PING, continuously, for the
>
> specified period) in order to consider it in S_DOWN state (Subjectively Down).

这个是说sentinel如果检测到master在&lt;milliseconds&gt;内都是unreachable，那个master就被认为是down了

`sentinel parallel-syncs <master_name> <replication_num> `， 这个配置是指定当有多少个Slave能够并行地对Master进行主从同步。

最后通过运行命令启动Sentinel即可：

> redis-sentinel /path/to/sentinel.conf

查看Sentinel的Info可以看到此时监控的master和Slave的信息。

尝试shutdown主机，可以发现slave将成为主机。
