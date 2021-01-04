### 一、AOF 介绍

AOF（append only file)通过保存 Redis 执行的写命令来保存数据库状态，从而完成数据同步，默认情况下 AOF 持久化是关闭的。

![](https://raw.githubusercontent.com/zhchenme/go/master/image/%E5%9F%BA%E7%A1%80/aof-process.png)

#### 1.1 AOF 相关配置

在介绍 AOF 持久化之前，先来了解一下 reis.conf 配置文件中关于 aof 配置的一些属性：

- appendonly：是否打开 AOF 持久化，默认 no
- appendfilename：AOF 文件名，默认 `appendonly.aof`
- appendfsync：AOF 刷盘同步策略
- no-appendfsync-on-rewrite：AOF 重写期间是否延迟文件同步，默认 no
- auto-aof-rewrite-percentage：当目前 aof 文件大小超过上一次重写时的 AOF 文件大小的百分之多少时会再次进行重写，默认 100
- auto-aof-rewrite-min-size：AOF 触发重写最小文件大小，默认 64mb

#### 1.2 持久化过程

AOF 持久化分为三个步骤：

 1. 命令追加：将写命令追加到 `aof_buf` 缓冲区
 2. 文件写入：将 `aof_buf` 缓冲区中的数据保存到内存缓冲区中，当内存缓冲区的空间被填满，或者超过了指定的超时时间，才将内存缓冲区的数据写到磁盘文件
 3. 文件同步：将内存缓冲区中的文件写到磁盘

`aof_buf` 缓冲区与内存缓冲区是两个不同的概念，有不同的含义：

  - aof_buf：用来缓存客户端的写命令
  - 内存缓冲区：用来缓存磁盘文件写入的数据

PS：文件写入与文件同步的区别可以理解为 Java IO 中 `flush()` 方法执行前与执行后文件的两种状态。

当 AOF 持久化处于打开状态时，服务器在执行完一个写命令后，不会立即写入到文件中，会先按照一定的协议格式将写命令追加到服务器的 `aof_buf` 缓冲区中。

Redis 服务器本身是事件驱动程序，在服务器每次结束一个事件循环之前，都会调用 `flushAppendOnlyFile` 函数判断是否需要把 `aof_buf` 缓冲区的内容写入和保存到 AOF 磁盘文件里。

`flushAppendOnlyFile` 函数的行为由服务器的 `appendfsync` 选项值决定，不同的值有不同的行为，默认是 `everysec`。

 - always：将 `aof_buf` 缓冲区的所有内容写入并同步到 AOF 文件
 - everysec：将 `aof_buf` 缓冲区的所有内容写入到 AOF 文件，如果上次同步 AOF 文件的时间超过 1s，则对 AOF 文件进行同步
 - no：将 `aof_buf` 缓存中的内容写到 AOF 文件，何时同步由操作系统决定

RDB 持久化可能会丢失部分数据，这个问题在 AOF 中仍然存在。当配置策略为 `always` 时，当出现意外故障宕机的情况下会可能会丢失一个事件循环所产生的数据，如果为 `everysec` 最多会丢失 1s 内的数据，至于 `no` 配置，数据丢失多少完全取决于操作系统。

#### 1.3 数据还原

AOF 文件里包含了重建数据库状态的所有写命令，当服务器启动时只要重新读取并执行相关的写命令就能将数据恢复到关闭之前的状态。

PS：由于 AOF 触发频率高于 RDB，当两种持久化策略都开启时，会优先使用 AOF 文件恢复数据库状态。

 1. 创建一个不带网络连接的伪客户端，执行 AOF 文件保存的写命令（因为载入 AOF 文件不需要通过网络连接处理请求，因此创建的是伪客户端）
 2. 从 AOF 文件中分析一条写命令
 3. 通过伪客户端执行写命令
 4. 重复步骤 3 和 4，直到 AOF 文件的写命令被处理完

### 二、AOF 文件重写

#### 2.1 重写介绍

Redis 如果写很频繁会导致 AOF 文件的体积越来越大，使用 AOF 还原数据所需要的时间就越多。频繁修改一个 key 的 value，其实对于 redis 服务器来说他只要记录最后一条写命令即可。

为了解决 AOF 文件体积过大的问题，Redis 提供了 AOF 文件重写功能，通过重写功能 Redis 服务器会创建一个新的 AOF 文件替代原有的 AOF 文件，新旧两个文件保存的数据库状态相同，但是新 AOF 文件不会保存冗余的写命令，所以新的 AOF 文件体积会比较小。

**AOF 重写并不是对旧的 AOF 进行读取、分析、写入操作，而是通过读取服务器当前的数据库状态实现。**

### 2.2 重写原理

![](https://raw.githubusercontent.com/zhchenme/go/master/image/%E5%9F%BA%E7%A1%80/aof-rewrite.png.png)

AOF 后台重写期间，服务器进程还会继续处理命令请求，如果新的命令对数据库状态进行修改，可能会导致新 AOF 文件与数据库状态不一致。

为了解决这个问题，redis 设置了一个 AOF 重写缓冲区，这个缓冲区在服务器创建子进程之后使用，当 Redis 服务器执行完一个写命令，会同时将这个写命令发送到 AOF 缓冲区和 AOF 重写缓冲区。

AOF 重写在一个子进程里执行，因此并不会阻塞服务器接收客户端的请求。子进程带有服务器进程数据的副本，使用子进程而不是线程，可以避免在使用锁的情况下，保证数据的安全性。

当子进程完成 AOF 重写，会向父进程发送一个信息，父进程在接收到信号后会调用信号处理函数，将 AOF 重写缓冲区的所有内容写入到新的 AOF 文件中，并完成 AOF 文件更名，覆盖现有的 AOF 文件，完成新旧文件替换。

### 三、other

知道了上面的原理，下面我们重点看一个配置参数 `no-appendfsync-on-rewrite`，含义是 AOF 重写期间是否阻塞 `appendfsync`。重写期间由于主进程执行 AOF 持久化，子进程执行 AOF 重写都需要操作磁盘，可能造成主进程在写 AOF 文件的时候出现阻塞的情形。

这个参数默认为 no，是最安全的方式，但是要忍受 `appendfsync` 持久化阻塞的问题，如果数据库中有大量数据，导致 AOF 重写时间过长，`appendfsync` 会一直被阻塞。

如果为 yes 表明 AOF 持久化并没有执行磁盘操作，只是将数据写入到 AOF 缓冲区，这是 Redis 挂了可能会丢数据。

### 参考
《Redis 设计与实现》 黄建宏 著 <br>
[Redis 持久化](http://www.redis.cn/topics/persistence.html) <br>
[一次非典型性 Redis 阻塞总结](https://liudanking.com/performance/%E4%B8%80%E6%AC%A1%E9%9D%9E%E5%85%B8%E5%9E%8B%E6%80%A7-redis-%E9%98%BB%E5%A1%9E%E6%80%BB%E7%BB%93/) <br>