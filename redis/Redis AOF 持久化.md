### 一、AOF

AOF（append only file)通过保存 Redis 执行的写命令来保存数据库状态，从而完成数据同步。默认情况下 AOF 持久化是关闭的。

AOF 持久化分为三个步骤：

 1. 命令追加
 2. 文件写入
 3. 文件同步

当 AOF 持久化处于打开状态时，服务器在执行完一个写命令后，会按照一定的协议格式将写命令追加到服务器的 `aof_buf` 缓冲区中。

Redis 服务器本身是事件驱动程序，所以在服务器每次结束一个事件循环之前，都会调用 `flushAppendOnlyFile` 函数判断是否需要把 `aof_buf` 缓冲区的内容写入和保存到 AOF 文件里。`flushAppendOnlyFile` 函数的行为由服务器的 `appendfsync` 选项值决定，不同的值有不同的行为，默认是 `everysec`。

 always：将 `aof_buf` 缓冲区的所有内容写入并同步到 AOF 文件
 everysec：将 `aof_buf` 缓冲区的所有内容写入到 AOF 文件，如果上次同步 AOF 文件的事件超过 1m 中，则对 AOF 文件进行同步
 no：将 `aof_buf` 缓存中的内容写到 AOF 文件，何时同步由操作系统决定



### 参考
《Redis 设计与实现》 黄建宏 著