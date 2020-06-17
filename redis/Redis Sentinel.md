Redis sentinel 本质是一个运行在特殊模式下的 Redis 服务器，一般由一个或多个实例组成 sentinel 系统，可以监视任意多个主服务器及这些主服务器下的所有从服务器。

### 一、sentinel.conf 文件详解

 - port 26379：sentinel 端口
 - sentinel monitor mymaster 127.0.0.1 6379 2：哨兵主要配置
    - master-name：自定义的主服务器名
    - ip：主服务器 ip
    - port：主服务器端口
    - quorum：主服务器客观下线的阈值
 - sentinel down-after-milliseconds mymaster 30000：节点响应哨兵最大响应时间，超过该时间，哨兵认为节点主观下线
 - sentinel parallel-syncs mymaster 1：
 - sentinel failover-timeout mymaster 180000：故障恢复时间
 - 

### 二、主观下线与客观下线

#### 2.1 主观下线

默认情况下，sentinel 会以每秒一次的频率向所有创建了命令连接的实例（主服务器、从服务器、其他 sentinel 实例）发送 `PING` 命令，病通过实例返回的 `PING` 命令的回复判断实例是否下线。

sentinel 的配置文件中 `down-after-milliseconds` 属性指定了判断服务器主观下线的时间长度，默认是 30s，在 30s 内如果服务器没有响应 sentinel 发送的 `PING` 命令，sentinel 会把该服务器标记为主观下线状态。

#### 2.2 客观下线

sentinel 将一个主服务器标记为主观下线之后，为了确认这个主服务器是否真的下线，会向监视该主服务器的其他 sentinel 进行询问，当该 sentinel 从其他 sentinel 接收到足够数量的以下线判断后，sentinel 就会将主服务器判定为客观下线，并对主服务器进行故障迁移操作。这个数量就是上面的 `quorum` 参数值。

PS：客观下线只针对主服务器

### sentinel 选举

