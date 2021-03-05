### 正向代理与反向代理

- 正向代理：代理的是客户端，比如我们使用的 VPN 软件等
- 反向代理：代理的是服务器，用来分发用户请求到不同服务器上

### Nginx 负载均衡策略

- 轮训（round_robin）：每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器 down 掉，能自动剔除
- IP 哈希（ip_hash）：每个请求按访问 ip 的 hash 结果分配，这样每个访客固定访问一个后端服务器，可以解决 session 共享的问题
- 最少连接（least_conn）：下一个请求将被分派到活动连接数量最少的服务器
- 权重（weight）：指定轮询几率，weight 和访问比率成正比
- url_hash：按访问 url 的hash结果来分配请求，使每个 url 定向到同一个后端服务器

### Master 与 Woker 进程

- Master：读取并验证配置文件 nginx.conf，管理 worker 进程
- Woker：每一个 Worker 进程都维护一个线程（避免线程切换），处理连接和请求，注意 Worker 进程的个数由配置文件决定，一般和 CPU 个数相关（有利于进程切换），配置几个就有几个 Worker 进程

Nginx 采用了 Linux 的 epoll 模型，epoll 模型基于事件驱动机制，它可以监控多个事件是否准备完毕，如果 OK，那么放入 epoll 队列中，这个过程是异步的。worker 只需要从 epoll 队列循环处理即可。

### 参考

[Nginx的五种负载均衡策略](http://www.openskill.cn/article/586) by chris <br>
[8分钟带你深入浅出搞懂Nginx](https://zhuanlan.zhihu.com/p/34943332) by 地球的外星人君 <br>
[前端想要了解的Nginx](https://juejin.im/post/5cae9de95188251ae2324ec3) by 快狗打车前端团队 <br>