### 正向代理与反向代理

- 正向代理：代理的是客户端，比如我们使用的 VPN 软件等
- 反向代理：代理的是服务器，用来分发用户请求到不同服务器上

### Nginx 负载均衡策略

- 轮训（round_robin）：每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器 down 掉，能自动剔除
- IP 哈希（ip_hash）：每个请求按访问 ip 的 hash 结果分配，这样每个访客固定访问一个后端服务器，可以解决 session 共享的问题
- 最少连接（least_conn）：下一个请求将被分派到活动连接数量最少的服务器
- 权重（weight）：指定轮询几率，weight 和访问比率成正比
- url_hash：按访问 url 的hash结果来分配请求，使每个 url 定向到同一个后端服务器



### 参考

(Nginx的五种负载均衡策略)[http://www.openskill.cn/article/586] by chris