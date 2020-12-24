### 一、时间轮介绍

之前公司内部搭建的延迟队列服务有用到时间轮，但是一直没有了解过它的实现原理。

最近有个和支付宝对接的项目，支付宝接口有流量控制，一定的时间内只允许 N 次接口调用，针对一些业务我们需要频繁调用支付宝开放平台接口，如果不对请求做限制，很容易触发流控告警。

为了避免这个问题，我们按照一定延迟规则将任务加载进时间轮内，通过时间轮的调度来实现接口异步调用。

很多开源框架都实现了时间轮算法，这里以 Netty 为例，看下 Netty 中时间轮是怎么实现的。

#### 1.1 快速入门

下面是一个 API 使用例子。

```java
@Slf4j
public class TimerWheelSamples {

    static final HashedTimerWheelInstance INSTANCE = HashedTimerWheelInstance.INSTANCE;

    public static void main(String[] args) throws IOException {

        INSTANCE.getWheelTimer().newTimeout(new TimerTaskInstance(), 3, TimeUnit.SECONDS);
        System.in.read();
    }


    static class TimerTaskInstance implements TimerTask {
        @Override
        public void run(Timeout timeout) {
            log.info("Hello world");
        }
    }

    enum HashedTimerWheelInstance {
        INSTANCE;
        private final HashedWheelTimer wheelTimer;

        HashedTimerWheelInstance() {
            wheelTimer = new HashedWheelTimer(r -> {
                Thread t = new Thread(r);
                t.setUncaughtExceptionHandler((t1, e) -> log.error(t1.getName() + e.getMessage()));
                t.setName("-HashedTimerWheelInstance-");
                return t;
            }, 100, TimeUnit.MILLISECONDS, 64);
        }

        public HashedWheelTimer getWheelTimer() {
            return wheelTimer;
        }
    }
}
```

上面的例子中我们自定义了一个 `HashedWheelTimer`，然后自定义了一个 `TimerTask`，将一个任务加载进时间轮，3s 后执行这个任务，怎么样是不是很简单。

在定义时间轮时建议按照业务类型进行区分，将时间轮定义为多个单例对象。

PS：因为时间轮是异步执行的，在任务执行之前 JVM 不能退出，所以 `System.in.read();` 这一行代码不能删除。

#### 1.2 原理图解

### 二、原理分析

#### 2.1 时间轮状态

时间轮有以下三种状态：

 - WORKER_STATE_INIT：初始化状态，此时时间轮内的工作线程还没有开启
 - WORKER_STATE_STARTED：运行状态，时间轮内的工作线程已经开启
 - WORKER_STATE_SHUTDOWN：终止状体，时间轮停止工作

状态转换如下，转换原理会在下面讲到：



#### 2.2 构造函数

```java
    public HashedWheelTimer(
            ThreadFactory threadFactory,
            long tickDuration, TimeUnit unit, int ticksPerWheel, boolean leakDetection,
            long maxPendingTimeouts) {

        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }
        if (tickDuration <= 0) {
            throw new IllegalArgumentException("tickDuration must be greater than 0: " + tickDuration);
        }
        if (ticksPerWheel <= 0) {
            throw new IllegalArgumentException("ticksPerWheel must be greater than 0: " + ticksPerWheel);
        }

        // 初始化时间轮数组，时间轮大小为大于等于 ticksPerWheel 的第一个 2 的幂，和 HashMap 类似
        wheel = createWheel(ticksPerWheel);
        // 取模用，用来定位数组中的槽
        mask = wheel.length - 1;

        // 为了保证精度，时间轮内的时间单位为纳秒
        long duration = unit.toNanos(tickDuration);

        // 时间轮内的时钟转动时间不宜太大也不宜太小
        if (duration >= Long.MAX_VALUE / wheel.length) {
            throw new IllegalArgumentException(String.format(
                    "tickDuration: %d (expected: 0 < tickDuration in nanos < %d",
                    tickDuration, Long.MAX_VALUE / wheel.length));
        }

        if (duration < MILLISECOND_NANOS) {
            logger.warn("Configured tickDuration {} smaller then {}, using 1ms.",
                        tickDuration, MILLISECOND_NANOS);
            this.tickDuration = MILLISECOND_NANOS;
        } else {
            this.tickDuration = duration;
        }

        // 创建工作线程
        workerThread = threadFactory.newThread(worker);


        // 非守护线程且 leakDetection 为 true 时检测内存是否泄漏
        leak = leakDetection || !workerThread.isDaemon() ? leakDetector.track(this) : null;

        // 初始化最大等待任务数
        this.maxPendingTimeouts = maxPendingTimeouts;

        // 如果创建的时间轮实例大于 64，打印错误日志，并且这个日志只会打印一次
        if (INSTANCE_COUNTER.incrementAndGet() > INSTANCE_COUNT_LIMIT &&
            WARNED_TOO_MANY_INSTANCES.compareAndSet(false, true)) {
            reportTooManyInstances();
        }
    }
```

构造函数中的参数相当重要，当自定义时间轮时，我们应该根据业务的范围设置合理的参数：

 - threadFactory：创建时间轮任务线程的工厂，通过这个工厂可以给我们的线程自定义一些属性(线程名、异常处理等)
 - tickDuration：时钟多长时间转动一次，值越小，时间轮精度越高
 - unit：`tickDuration` 的单位
 - ticksPerWheel：时间轮数组大小
 - leakDetection：是否检测内存泄漏
 - maxPendingTimeouts：时间轮内最大等待的任务数

时间轮的时钟转动时长应该根据业务设置恰当的值，如果设置的过大，可能导致任务触发时间不准确。如果设置的过小，时间轮转动频繁，任务少的情况下加载不到任务，属于一直空转的状态，会占用 CPU 线程资源。

为了防止时间轮占用过多的 CPU 资源，当创建的时间轮对象大于 64 时会以日志的方式提示。

构造函数中只是初始化了轮线程，并没有开启，当第一次往时间轮内添加任务时，线程才会开启。

#### 2.3 往时间轮内添加任务

```java
    @Override
    public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (unit == null) {
            throw new NullPointerException("unit");
        }

        // 等待的任务数 +1
        long pendingTimeoutsCount = pendingTimeouts.incrementAndGet();

        // 如果时间轮内等待的任务数大于最大值，任务会被抛弃
        if (maxPendingTimeouts > 0 && pendingTimeoutsCount > maxPendingTimeouts) {
            pendingTimeouts.decrementAndGet();
            throw new RejectedExecutionException("Number of pending timeouts ("
                + pendingTimeoutsCount + ") is greater than or equal to maximum allowed pending "
                + "timeouts (" + maxPendingTimeouts + ")");
        }

        // 开启时间轮内的线程
        start();

        // 计算当前添加任务的执行时间
        long deadline = System.nanoTime() + unit.toNanos(delay) - startTime;

        // Guard against overflow.
        if (delay > 0 && deadline < 0) {
            deadline = Long.MAX_VALUE;
        }
        // 将任务加入队列
        HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
        timeouts.add(timeout);
        return timeout;
    }
```

任务会先保存在队列中，当时间轮的时钟转动时才会判断是否将队列中的任务加载进时间轮。

```java
    public void start() {
        switch (WORKER_STATE_UPDATER.get(this)) {
            case WORKER_STATE_INIT:
                // 这里存在并发，通过 CAS 操作保证最终只有一个线程能开启时间轮的工作线程
                if (WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_INIT, WORKER_STATE_STARTED)) {
                    workerThread.start();
                }
                break;
            case WORKER_STATE_STARTED:
                break;
            case WORKER_STATE_SHUTDOWN:
                throw new IllegalStateException("cannot be started once stopped");
            default:
                throw new Error("Invalid WorkerState");
        }

        while (startTime == 0) {
            try {
                // startTimeInitialized 是一个 CountDownLatch，目的是为了保证工作线程的 startTime 属性初始化
                startTimeInitialized.await();
            } catch (InterruptedException ignore) {
                // Ignore - it will be ready very soon.
            }
        }
    }
```

这里通过 CAS 加锁的方式保证线程安全，避免多次开启。

工作线程开启后，`start()` 方法会被阻塞，等工作线程的 `startTime` 属性初始化完成后才被唤醒。为什么只有等 `startTime` 初始化后才能继续执行呢？因为上面的 `newTimeout` 方法在线程开启后，需要计算当前添加进来任务的执行时间，而这个执行时间是根据 `startTime` 计算的。

#### 2.4 时间轮调度

```java
        @Override
        public void run() {
            // 初始化 startTime.
            startTime = System.nanoTime();
            if (startTime == 0) {
                // We use 0 as an indicator for the uninitialized value here, so make sure it's not 0 when initialized.
                startTime = 1;
            }

            // 用来唤醒被阻塞的 HashedWheelTimer#start() 方法，保证 startTime 初始化
            startTimeInitialized.countDown();

            do {
                // 时钟拨动
                final long deadline = waitForNextTick();
                if (deadline > 0) {
                    int idx = (int) (tick & mask);
                    // 处理过期的任务
                    processCancelledTasks();
                    HashedWheelBucket bucket =
                            wheel[idx];
                    // 将任务加载进时间轮
                    transferTimeoutsToBuckets();
                    // 执行当前时间轮槽内的任务
                    bucket.expireTimeouts(deadline);
                    tick++;
                }
            } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);

            // 时间轮关闭，将还未执行的任务以列表的形式保存到 unprocessedTimeouts 集合中，在 stop 方法中返回出去
            // 还未执行的任务可能会在两个地方，一：时间轮数组内，二：队列中，这两种情况都要处理
            for (HashedWheelBucket bucket: wheel) {
                bucket.clearTimeouts(unprocessedTimeouts);
            }
            for (;;) {
                HashedWheelTimeout timeout = timeouts.poll();
                if (timeout == null) {
                    break;
                }
                if (!timeout.isCancelled()) {
                    unprocessedTimeouts.add(timeout);
                }
            }
            // 处理过期的任务
            processCancelledTasks();
        }
```

时间轮每拨动一次 `tick` 就会 +1，根据这个值与(时间轮数组长度 - 1)进行 `&` 运算，可以定位时间轮数组内的槽。因为 `tick` 值一直在增加，所以时间轮数组看起来就像一个不断循环的圆。

 - 先初始化 `startTime` 值，因为后面任务执行的时间是根据 `startTime` 计算的
 - 时钟拨动，如果时间未到，则 `sleep` 一会儿
 - 处理过期的任务
 - 将任务加载进时间轮
 - 执行当前时钟对应时间轮内的任务
 - 时间轮关闭，将所有未执行的任务封装到 `unprocessedTimeouts` 集合中，在 `stop` 方法中返回出去
 - 处理过期的任务

 上面简单罗列了下 `run` 方法的大概执行步骤，下面是具体方法的分析。

#### 2.5 时钟拨动

如果时间轮设置的 `tickDuration` 为 100ms 拨动一次，当时钟拨动一次后，应该计算下一次时钟拨动的时间，如果还没到就 `sleep` 一会儿，等到拨动时间再醒来。

```java
        private long waitForNextTick() {
            // 计算时钟下次拨动的相对时间
            long deadline = tickDuration * (tick + 1);

            for (;;) {
                // 获取当前时间的相对时间
                final long currentTime = System.nanoTime() - startTime;
                // 计算距离时钟下次拨动的时间
                // 这里之所以加 999999 后再除 10000000, 是为了保证足够的 sleep 时间
                // 例如：当 deadline - currentTime = 2000002 的时候，如果不加 999999，则只睡了 2ms
                // 而 2ms 其实是未到达 deadline 时间点的，所以为了使上述情况能 sleep 足够的时间，加上 999999 后，会多睡 1ms
                long sleepTimeMs = (deadline - currentTime + 999999) / 1000000;

                // <=0 说明可以拨动时钟了
                if (sleepTimeMs <= 0) {
                    if (currentTime == Long.MIN_VALUE) {
                        return -Long.MAX_VALUE;
                    } else {
                        return currentTime;
                    }
                }

                
                // 这里是为了兼容 Windows 平台，因为 Windows 平台的调度最小单位为 10ms，如果不是 10ms 的倍数，可能会引起 sleep 时间不准确
                // See https://github.com/Netty/Netty/issues/356
                if (PlatformDependent.isWindows()) {
                    sleepTimeMs = sleepTimeMs / 10 * 10;
                }

                try {
                    // sleep 到下次时钟拨动
                    Thread.sleep(sleepTimeMs);
                } catch (InterruptedException ignored) {
                    if (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_SHUTDOWN) {
                        return Long.MIN_VALUE;
                    }
                }
            }
        }
```

如果时间不到就 `sleep` 等待一会儿，为了使任务时钟准确，可以从上面的代码中看出 Netty 做了一些优化，如果 `sleepTimeMs` 的计算，Windows 平台的处理等。

#### 2.6 将任务从队列加载进时间轮

```java
        private void transferTimeoutsToBuckets() {
            
            // 一次最多只处理队列中的 100000 个任务
            for (int i = 0; i < 100000; i++) {
                HashedWheelTimeout timeout = timeouts.poll();
                if (timeout == null) {
                    // all processed
                    break;
                }
                // 过滤已经取消的任务
                if (timeout.state() == HashedWheelTimeout.ST_CANCELLED) {
                    // Was cancelled in the meantime.
                    continue;
                }
                // 计算当前任务到执行还需要经过几次时钟拨动
                // 假设时间轮数组大小是 10，calculated 为 12，需要时间轮转动一圈加两次时钟拨动后后才能执行这个任务，因此还需要计算一下圈数
                long calculated = timeout.deadline / tickDuration;
                // 计算当前任务到执行还需要经过几圈时钟拨动
                timeout.remainingRounds = (calculated - tick) / wheel.length;
                // 有的任务可能在队列里很长时间，时间过期了也没有被调度，将这种情况的任务放在当前轮次内执行
                final long ticks = Math.max(calculated, tick); // Ensure we don't schedule for past.
                // 计算任务在时间轮数组中的槽
                int stopIndex = (int) (ticks & mask);
                HashedWheelBucket bucket = wheel[stopIndex];
                // 将任务放到时间轮的数组中，多个任务可能定位时间轮的同一个槽，这些任务通过以链表的形式链接
                bucket.addTimeout(timeout);
            }
        }

        void addTimeout(HashedWheelTimeout timeout) {
            assert timeout.bucket == null;
            // 任务构成双向链表
            timeout.bucket = this;
            if (head == null) {
                head = tail = timeout;
            } else {
                tail.next = timeout;
                timeout.prev = tail;
                tail = timeout;
            }
        }        
```

在上面也提到过，任务刚加进来不会立即到时间轮中去，而是暂时保存到一个队列中，当时间轮时钟拨动时，会将任务从队列中加载进时间轮内。

时间轮每次最大处理 100000 个任务，因为任务的执行时间是用户自定义的，所以需要计算任务到执行需要经过多少次时钟拨动，并计算时间轮拨动的圈数。接着将任务加载进时间轮对应的槽内，可能有多个任务经过 hash 计算后定位到同一个槽，这些任务会以双向链表的结构保存，有点类似 `HashMap` 处理碰撞的情况。

#### 2.7 执行任务

```java
        public void expireTimeouts(long deadline) {
            HashedWheelTimeout timeout = head;

            // process all timeouts
            while (timeout != null) {
                HashedWheelTimeout next = timeout.next;
                // 任务执行的圈数 > 0，表示任务还需要经过 remainingRounds 圈时钟循环才能执行
                if (timeout.remainingRounds <= 0) {
                    // 从链表中移除当前任务，并返回链表中下一个任务
                    next = remove(timeout);
                    if (timeout.deadline <= deadline) {
                        // 执行任务
                        timeout.expire();
                    } else {
                        // The timeout was placed into a wrong slot. This should never happen.
                        throw new IllegalStateException(String.format(
                                "timeout.deadline (%d) > deadline (%d)", timeout.deadline, deadline));
                    }
                } else if (timeout.isCancelled()) {
                    // 过滤取消的任务
                    next = remove(timeout);
                } else {
                    // 圈数 -1
                    timeout.remainingRounds --;
                }
                timeout = next;
            }
        }

        public void expire() {
            // 任务状态校验
            if (!compareAndSetState(ST_INIT, ST_EXPIRED)) {
                return;
            }

            try {
                task.run(this);
            } catch (Throwable t) {
                if (logger.isWarnEnabled()) {
                    logger.warn("An exception was thrown by " + TimerTask.class.getSimpleName() + '.', t);
                }
            }
        }
```

时间轮槽内的任务以链表形式存储，这些任务执行的时间可能会不一样，有的在当前时钟执行，有的在下一圈或者下两圈对应的时钟执行。当任务在当前时钟执行时，需要将这个任务从链表中删除，重新维护链表关系。

上面就是时间轮运行的基本原理了。

#### 2.8 终止时间轮

```java
    @Override
    public Set<Timeout> stop() {
        // 终止时间轮的线程不能是时间轮的工作线程
        if (Thread.currentThread() == workerThread) {
            throw new IllegalStateException(
                    HashedWheelTimer.class.getSimpleName() +
                            ".stop() cannot be called from " +
                            TimerTask.class.getSimpleName());
        }
        // 将时间轮的状态修改为 WORKER_STATE_SHUTDOWN，这里有两种情况可能会更新失败
        // 一：时间轮是 WORKER_STATE_INIT 状态，表明时间轮从创建到终止一直没有任务进来
        // 二：时间轮是 WORKER_STATE_STARTED 状态，多个线程尝试终止时间轮，只有一个操作成功
        if (!WORKER_STATE_UPDATER.compareAndSet(this, WORKER_STATE_STARTED, WORKER_STATE_SHUTDOWN)) {
            // 代码走到这里，时间轮只能是两种状态中的一个，WORKER_STATE_INIT 和 WORKER_STATE_SHUTDOWN
            // 为 WORKER_STATE_INIT 表示时间轮没有任务，因此不用返回未处理的任务，但是需要将时间轮实例 -1
            // 为 WORKER_STATE_SHUTDOWN 表示是 CAS 操作失败，什么都不用做，因为 CAS 成功的线程会处理
            if (WORKER_STATE_UPDATER.getAndSet(this, WORKER_STATE_SHUTDOWN) != WORKER_STATE_SHUTDOWN) {
                // 时间轮实例对象 -1
                INSTANCE_COUNTER.decrementAndGet();
                if (leak != null) {
                    boolean closed = leak.close(this);
                    assert closed;
                }
            }
            // CAS 操作失败，或者时间轮没有处理过任务，返回空的任务列表
            return Collections.emptySet();
        }

        try {
            boolean interrupted = false;
            while (workerThread.isAlive()) {
                // 中断时间轮工作线程
                workerThread.interrupt();
                try {
                    // 终止时间轮的线程等待时间轮工作线程 100ms，这个过程主要是为了时间轮工作线程处理未执行的任务
                    workerThread.join(100);
                } catch (InterruptedException ignored) {
                    interrupted = true;
                }
            }

            if (interrupted) {
                Thread.currentThread().interrupt();
            }
        } finally {
            INSTANCE_COUNTER.decrementAndGet();
            if (leak != null) {
                boolean closed = leak.close(this);
                assert closed;
            }
        }
        // 返回未处理的任务
        return worker.unprocessedTimeouts();
    }
```

当终止时间轮时，时间轮状态有两种情况：

 - WORKER_STATE_INIT：时间轮初始化，前面我们说过，当初始化时间轮对象时并不会立即开启时间轮工作线程，而是第一次添加任务时才开启，为 `WORKER_STATE_INIT` 表示时间轮没有处理过任务
 - `WORKER_STATE_STARTED`：时间轮在工作，这里也有两种情况，存在并发与不存在并发，如果多个线程都尝试终止时间轮，肯定只能有一个成功

时间轮停止运行后会将未执行的任务返回出去，至于怎么处理这些任务，由业务方自己定义，这个流程和线程池的 `shutdownNow` 方法是类似的。

如果时间轮在运行，怎么才能获取到未执行的任务呢，答案就在上面的 `run()` 方法中，如果时间轮处于非运行状态，会把时间轮数组与队列中未执行且未取消的任务保存到 `unprocessedTimeouts` 集合中。而终止时间轮成功的线程只需要等待一会儿即可，这个等待是通过 `workerThread.join(100);` 实现的。


取消时间轮内的任务相对比较简单，这里就不概述了，想要了解的自行查看即可。

### 三、总结

这里以问答的形式进行总结，大家也可以看下这些问题，自己能不能很好的回答出来？

#### 3.1 时间轮是不是在初始化完成后就启动了？

A：不是，初始化完成时间轮的状态是 `WORKER_STATE_INIT`，此时时间轮内的工作线程还没有运行，只有第一次往时间轮内添加任务时，才会开启时间轮内的工作线程。时间轮线程开启后会初始化 `startTime`，任务的执行时间会根据这个字段计算，而且时间轮中时间的概念是相对的。

#### 3.2 如果时间轮内还有任务未执行，服务重启了怎么办？

A：时间轮内的任务都在内存中，服务重启数据肯定都丢了，所以当服务重启时需要业务方自己做兼容处理。

#### 3.3 如何自定义合适的时间轮参数？

自定义时间轮时有两个比较重要的参数需要我们注意：
 
 - tickDuration：时钟拨动频率，假设一个任务在 10s 后执行，`tickDuration` 设置为 3min 那肯定是不行的，`tickDuration` 值越小，任务触发的精度越高，但是没有任务时，工作线程会一直自旋尝试从队列中拿任务，比较消耗 CPU 资源
 - ticksPerWheel：时间轮数组大小，假设当时间轮时钟拨动时，有 10000 个任务处理，但是我们定义时间轮数组的大小为 8，这时平均一个时间轮槽内有 1250 个任务，如果这 1250 个任务都在当前时钟执行，任务执行是同步的，由于每个任务执行都会消耗时间，可能会导致后面的任务触发时间不准确。反之如果数组长度设置的过大，任务比较少的情况下，时间轮数组很多槽都是空的

所以当使用自定义时间轮时，一定要评估自己的业务后再设置参数。

#### 3.4 Netty 的时间轮有什么缺陷？

Netty 中的时间轮是通过单线程实现的，如果在执行任务的过程中出现阻塞，会影响后面任务执行。除此之外，Netty 中的时间轮并不适合创建延迟时间跨度很大的任务，比如往时间轮内丢成百上千个任务并设置 10 天后执行，这样可能会导致链表过长 `round` 值很大，而且这些任务在执行之前会一直占用内存。

#### 3.5 时间轮要设置成单例的吗？

强烈建议按照业务模块区分，每个模块都创建一个单例的时间轮对象。在上面的代码中我们看到了，时间轮最大只允许创建 64 个。如果时间轮是非单例对象，那时间轮算法完全就失去了作用。

#### 3.6 时间轮与 ScheduledExecutorService 的区别

`ScheduledExecutorService` 中的任务维护了一个堆，当有大量任务时，需要调整堆结构导致性能下降，而时间轮通过时钟调度，可以不受任务量的限制。

当任务量比较少时时间轮会一直自旋空转拨动时钟，相比 `ScheduledExecutorService` 会占用一定 CPU 资源。

#### 参考

[netty源码解读之时间轮算法实现-HashedWheelTimer](https://zacard.net/2016/12/02/netty-hashedwheeltimer/)
[HashedWheelTimer 使用及源码分析
创建](https://www.javadoop.com/post/HashedWheelTimer)
[定时器的几种实现方式](https://www.cnkirito.moe/timer/)