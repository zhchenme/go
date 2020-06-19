### 一、sentinel 概述

Redis sentinel 本质是一个运行在特殊模式下的 Redis 服务器，一般由一个或多个实例组成 sentinel 系统，可以监视任意多个主服务器及这些主服务器下的所有从服务器。

#### 1.1 sentinel 的作用

 - 监控（Monitoring）：Sentinel 会不断地检查你的主服务器和从服务器是否运作正常
 - 提醒（Notification）：当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知
 - 自动故障迁移（Automatic failover）：当一个主服务器不能正常工作时， Sentinel 会开始一次自动故障迁移操作， 它会将失效主服务器的其中一个从服务器升级为新的主服务器， 并让失效主服务器的其他从服务器改为复制新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器

#### 1.2 sentinel 配置文件详解

 - port 26379：sentinel 端口
 - daemonize no：非守护进程
 - sentinel monitor mymaster 127.0.0.1 6379 2
    - master-name：自定义的主服务器名
    - ip：主服务器 ip
    - port：主服务器端口
    - quorum：主服务器客观下线的阈值
 - sentinel down-after-milliseconds mymaster 30000：节点响应哨兵最大响应时间，超过该时间，哨兵认为节点主观下线
 - sentinel parallel-syncs mymaster 1：指定了在执行故障转移时， 最多可以有多少个从服务器同时对新的主服务器进行同步， 这个数字越小， 完成故障转移所需的时间就越长
 - sentinel failover-timeout mymaster 180000：故障恢复时间

### 二、主观下线与客观下线

#### 2.1 主观下线

默认情况下，sentinel 会以每秒一次的频率向所有创建了命令连接的实例（主服务器、从服务器、其他 sentinel 实例）发送 `PING` 命令，病通过实例返回的 `PING` 命令的回复判断实例是否下线。

sentinel 的配置文件中 `down-after-milliseconds` 属性指定了判断服务器主观下线的时间长度，默认是 30s，在 30s 内如果服务器没有响应 sentinel 发送的 `PING` 命令，sentinel 会把该服务器标记为主观下线状态。

如果一个主服务器被标记为主观下线， 那么正在监视这个主服务器的所有 Sentinel 要以每秒一次的频率确认主服务器的确进入了主观下线状态。

#### 2.2 客观下线

sentinel 将一个主服务器标记为主观下线之后，为了确认这个主服务器是否真的下线，会向监视该主服务器的其他 sentinel 进行询问，当该 sentinel 从其他 sentinel 接收到足够数量的以下线判断后，sentinel 就会将主服务器判定为客观下线，并对主服务器进行故障迁移操作。这个数量就是上面的 `quorum` 参数值。

在一般情况下， 每个 Sentinel 会以每 10 秒一次的频率向它已知的所有主服务器和从服务器发送 `INFO [section]` 命令。 当一个主服务器被 Sentinel 标记为客观下线时， Sentinel 向下线主服务器的所有从服务器发送 `INFO [section]` 命令的频率会从 10 秒一次改为每秒一次。

PS：客观下线只针对主服务器

### 三、自动发现 sentinel 和从服务器

Sentinel 可以通过发布与订阅功能来自动发现正在监视相同主服务器的其他 Sentinel ， 这一功能是通过向频道 `__sentinel__:hello` 发送信息来实现的。与此类似，Sentinel 可以通过询问主服务器来获得所有从服务器的信息。

 - 每个 Sentinel 会以每两秒一次的频率， 通过发布与订阅功能， 向被它监视的所有主服务器和从服务器的 `__sentinel__:hello` 频道发送一条信息， 信息中包含了 Sentinel 的 IP 地址、端口号和运行 ID （runid）
 - 每个 Sentinel 都订阅了被它监视的所有主服务器和从服务器的 `__sentinel__:hello` 频道， 查找之前未出现过的 sentinel。 当一个 Sentinel 发现一个新的 Sentinel 时， 它会将新的 Sentinel 添加到一个列表中， 这个列表保存了 Sentinel 已知的， 监视同一个主服务器的所有其他 Sentinel
 - Sentinel 发送的信息中还包括完整的主服务器当前配置。 如果一个 Sentinel 包含的主服务器配置比另一个 Sentinel 发送的配置要旧， 那么这个 Sentinel 会立即升级到新配置上

### 四、选举领头 sentinel

当主服务器被判断为客观下线时，监视这个主服务器的各个 sentinel 会进行协商，选举出一个领头 sentinel，由零头的 sentinel 进行后续的故障恢复工作。sentinel 选举是公平的，任何一个 sentinel 都可能成为领头。

领头 sentinel 选举采用的 Raft 算法，具体的算法可以参考最后面给出的链接，具体步骤如下：



当 Sentinel 接收到一个新的配置， 或者当领头 Sentinel 为主服务器创建一个新的配置时， 这个配置会与配置纪元一起被保存到磁盘里面，这意味着停止和重启 Sentinel 进程都是安全的。

#### 五、故障恢复

### 参考

[redisdoc](http://redisdoc.com/topic/sentinel.html#id3)
[Raft协议实战之Redis Sentinel的选举Leader源码解析](https://cloud.tencent.com/developer/article/1021467)
[一文搞懂 Raft 算法 ](https://www.cnblogs.com/xybaby/p/10124083.html)
[Raft 算法理解神器 ](http://thesecretlivesofdata.com/raft/)
