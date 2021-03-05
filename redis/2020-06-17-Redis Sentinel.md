### 一、Sentinel 概述

Redis Sentinel 本质是一个运行在特殊模式下的 Redis 服务器，一般由一个或多个实例组成 Sentinel 系统，可以监视任意多个主服务器及这些主服务器下的所有从服务器。

#### 1.1 Sentinel 的作用

 - 监控（Monitoring）：Sentinel 会不断地检查你的主服务器和从服务器是否运作正常
 - 提醒（Notification）：当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知
 - 自动故障迁移（Automatic failover）：当一个主服务器不能正常工作时， Sentinel 会开始一次自动故障迁移操作， 它会将失效主服务器的其中一个从服务器升级为新的主服务器， 并让失效主服务器的其他从服务器改为复制新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器

#### 1.2 Sentinel 配置文件详解

 - port 26379：Sentinel 端口
 - daemonize no：非守护进程
 - Sentinel monitor mymaster 127.0.0.1 6379 2
    - master-name：自定义的主服务器名
    - ip：主服务器 ip
    - port：主服务器端口
    - quorum：主服务器客观下线的阈值
 - Sentinel down-after-milliseconds mymaster 30000：节点响应哨兵最大响应时间，超过该时间，哨兵认为节点主观下线
 - Sentinel parallel-syncs mymaster 1：指定了在执行故障转移时， 最多可以有多少个从服务器同时对新的主服务器进行同步， 这个数字越小， 完成故障转移所需的时间就越长
 - Sentinel failover-timeout mymaster 180000：故障恢复超时时间

### 二、主观下线与客观下线

#### 2.1 主观下线

默认情况下，Sentinel 会以每秒一次的频率向所有创建了命令连接的实例（主服务器、从服务器、其他 Sentinel 实例）发送 `PING` 命令，并通过实例返回的 `PING` 命令的回复判断实例是否下线。

Sentinel 的配置文件中 `down-after-milliseconds` 属性指定了判断服务器主观下线的时间长度，默认是 30s，在 30s 内如果服务器没有响应 Sentinel 发送的 `PING` 命令，Sentinel 会把该服务器标记为主观下线状态。

如果一个主服务器被标记为主观下线， 那么正在监视这个主服务器的所有 Sentinel 要以每秒一次的频率确认主服务器的确进入了主观下线状态。

#### 2.2 客观下线

Sentinel 将一个主服务器标记为主观下线之后，为了确认这个主服务器是否真的下线，会向监视该主服务器的其他 Sentinel 进行询问，当该 Sentinel 从其他 Sentinel 接收到足够数量的已下线判断后，Sentinel 就会将主服务器判定为客观下线，并对主服务器进行故障迁移操作。这个数量就是上面的 `quorum` 参数值。

在一般情况下， 每个 Sentinel 会以每 10 秒一次的频率向它已知的所有主服务器和从服务器发送 `INFO [section]` 命令。 当一个主服务器被 Sentinel 标记为客观下线时， Sentinel 向下线主服务器的所有从服务器发送 `INFO [section]` 命令的频率会从 10 秒一次改为每秒一次。

PS：客观下线只针对主服务器

### 三、自动发现 Sentinel 和从服务器

Sentinel 可以通过发布与订阅功能来自动发现正在监视相同主服务器的其他 Sentinel ， 这一功能是通过向频道 `__Sentinel__:hello` 发送信息来实现的。与此类似，Sentinel 可以通过询问主服务器来获得所有从服务器的信息。

 - 每个 Sentinel 会以每两秒一次的频率， 通过发布与订阅功能， 向被它监视的所有主服务器和从服务器的 `__Sentinel__:hello` 频道发送一条信息， 信息中包含了 Sentinel 的 IP 地址、端口号和运行 ID （runid）
 - 每个 Sentinel 都订阅了被它监视的所有主服务器和从服务器的 `__Sentinel__:hello` 频道， 查找之前未出现过的 Sentinel。 当一个 Sentinel 发现一个新的 Sentinel 时， 它会将新的 Sentinel 添加到一个列表中， 这个列表保存了 Sentinel 已知的， 监视同一个主服务器的所有其他 Sentinel
 - Sentinel 发送的信息中还包括完整的主服务器当前配置。 如果一个 Sentinel 包含的主服务器配置比另一个 Sentinel 发送的配置要旧， 那么这个 Sentinel 会立即升级到新配置上

### 四、选举领头 Sentinel

当主服务器被判断为客观下线时，监视这个主服务器的各个 Sentinel 会进行协商，选举出一个领头 Sentinel，由领头的 Sentinel 进行后续的故障恢复工作。Sentinel 选举是公平的，切遵从先到先来原则，任何一个 Sentinel 都可能成为领头。

领头 Sentinel 选举采用的 Raft 算法，具体的算法可以参考最后面给出的链接，Redis Sentinel 选举步骤如下：
 
 1. 某个 Sentinel 认定 master 客观下线后，该 Sentinel 会先看看自己有没有投过票，如果自己已经投过票给其他 Sentinel 了，在 2 倍故障转移的超时时间自己就不会成为 Leader
 2. 如果该 Sentinel 还没投过票，那么它就成为 Candidate，成为 Candidate，Sentinel 需要完成几件事情 
    - 更新故障转移状态为 start
    - 当前 epoch（纪元）加 1，相当于进入一个新 term
    - 更新自己的超时时间为当前时间随机加上一段时间，随机时间为 1s 内的随机毫秒数
    - 向其他节点发送 `is-master-down-by-addr` 命令请求投票，命令会带上自己的 epoch
    - 给自己投一票
 3. 设置领头 Sentinel 遵从先到先来原则，最先向目标 Sentinel 发送命令请求的 Sentinel 将被目标 Sentinel 选举为局部领头 Sentinel，之后接收到的请求都会被拒绝
 4. 源 Sentinel 收到目标 Sentinel 的回复之后，会检查传来的 `leader_epoch`、`leader_runid` 参数值与自己是否一致，如果一致则表示目标 Sentinel 将源自己设置成了局部领头 Sentinel。Candidate 会不断的统计自己的票数，直到他发现认同他成为 Leader 的票数超过一半（而且超过它配置的 quorum， todo 待验证）
 5. 如果在一个 epoch 内，没有一个 Candidate 获得更多的票数。那么等待超过 2 倍故障转移的超时时间后，Candidate 增加 epoch 重新投票
 6. 如果某个 Candidate 获得超过一半票数而且超过它配置的 quorum，那么它就成为了 Leader
 7. Leader 并不会把自己成为 Leader 的消息发给其他 Sentinel。其他 Sentinel 等待 Leader 从 slave 选出 master 后，检测到新的 master 正常工作后，会去掉客观下线的标识，从而不需要进入故障转移流程

Sentinel 的个数最好设置成奇数个，避免在一个纪元内出现平票的情况，而第一个发现主服务器客观下线的 Sentinel 往往能够成为领头 Sentinel。

当 Sentinel 接收到一个新的配置，或者当领头 Sentinel 为主服务器创建一个新的配置时，这个配置会与配置纪元一起被保存到磁盘里面，这意味着停止和重启 Sentinel 进程都是安全的。

### 五、故障恢复

选举出领头 Sentinel 之后，领头 Sentinel 将负责故障迁移工作：

 1. 在所有从服务器里选择一个成为主服务器
 2. 让其他的从服务器复制新的主服务器
 3. 将已下线的主服务器设置为新的主服务器的从服务器，当旧的主服务器重新上线时，会成为新的主服务器的从服务器

从服务器成为主服务器将经过以下规则过滤：

 1. 在失效主服务器属下的从服务器当中，那些被标记为主观下线、已断线、或者最后一次回复领头 Sentinel `INFO` 命令的时间大于五秒钟的从服务器都会被淘汰
 2. 在失效主服务器属下的从服务器当中，那些与失效主服务器连接断开的时长超过 `down-after-milliseconds` 选项指定的时长十倍的从服务器都会被淘汰
 3. 领头 Sentinel 将根据剩余的从服务器中选择优先级较高的，如果存在多个优先级较高的从服务器，将选出复制偏移量（replication offset）最大的那个从服务器作为新的主服务器，如果复制偏移量不可用，或者从服务器的复制偏移量相同，那么带有最小运行 ID 的那个从服务器成为新的主服务器

### 参考

[redisdoc](http://redisdoc.com/topic/Sentinel.html#id3) <br>
[Raft 协议实战之 Redis Sentinel 的选举 Leader 源码解析 ](https://cloud.tencent.com/developer/article/1021467) <br>
[一文搞懂 Raft 算法 ](https://www.cnblogs.com/xybaby/p/10124083.html) <br>
[Raft 算法理解神器 ](http://thesecretlivesofdata.com/raft/) <br>
[Raft 一致性算法](https://lotabout.me/2019/Raft-Consensus-Algorithm/)
