### 一、负数二进制三码表示形式

* 原码：原码就是符号位加上真值的绝对值, 即用第一位表示符号, 其余位表示值
* 反码：负数的反码是在其原码的基础上, 符号位不变，其余各个位取反
* 补码：负数的补码是在其原码的基础上, 符号位不变, 其余各位取反, 最后+1 (即在反码的基础上+1)

正数三码合一，下面是一个负数的例子：

-1 的二进制原码 `10000000 00000000 00000000 00000001` <br>
-1 的二进制反码 `11111111 11111111 11111111 11111110` <br>
-1 的二进制补码 `11111111 11111111 11111111 11111111` <br>



### 二、SnowFlake 介绍

![](https://image-static.segmentfault.com/350/263/350263808-59c2254083397)

 - 1 位，表示符号最高位固定是 0,为 1 表示负数，因此该情况不存在
 - 41 位，用来记录时间戳（毫秒）
 - 41 位可以表示 `2 << 41 - 1` 个数字，转化成单位年则是 (2 << 41 - 1)/(1000 * 60 * 60 * 24 * 365) = 69 年
 - 10 位，用来记录工作机器 id。包括 5 位 datacenterId 和 5 位 workerId 5 位（bit）,可以表示的最大正整数是 `2 << 5 − 1`，即可以用 0、1、2、3、....31 这 32 个数字，来表示不同的 datecenterId 或 workerId
 - 12 位，序列号，用来记录同毫秒内产生的不同 id。12 位（bit）可以表示的最大正整数是 `2 << 12 - 1`，即可以用 0、1、2、3、....4095 这 4096 个数字，来表示同一机器同一时间截（毫秒) 内产生的 4096 个 ID 序号，1 秒能生成 400W 条 id

SnowFlake 依赖于时钟，因此可以保证 id 是递增的，但是如果时间回拨或者是分布式环境下机器分布于不同的时区，也可能会造成 id 重复。

根据自己的业务场景，可以对上面的 64 位进行自定义划分，如果当前的业务规模下并发量比较小，可以将序列号的位数适当调小，避免 id 的浪费。

### 三、SnowFlake 算法解读

相关代码：

``` java
public class SnowFlake {

    /**
     * 开始时间截 (2018-01-01)
     */
    private final long startTimestamp = 1514736000000L;

    private final int totalBit = 72;

    /**
     * 机器 id 所占的位数
     */
    private final long workerIdBits = 5L;

    /**
     * 数据标识 id 所占的位数
     */
    private final long dataCenterIdBits = 5L;

    /**
     * 支持的最大机器 id，结果是 31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数)
     */
    private final long maxWorkerId = ~(-1L << workerIdBits);

    /**
     * 支持的最大数据标识 id，结果是 31
     */
    private final long maxDataCenterId =~(-1L << dataCenterIdBits);

    /**
     * 序列在 id 中占的位数
     */
    private final long sequenceBits = 12L;

    /**
     * 机器 ID 向左移 12 位
     */
    private final long workerIdShift = sequenceBits;

    /**
     * 数据标识 id 向左移 17 位(12 + 5)
     */
    private final long dataCenterIdShift = sequenceBits + workerIdBits;

    /**
     * 时间截向左移 22 位(5 + 5 + 12)
     */
    private final long timestampLeftShift = sequenceBits + workerIdBits + dataCenterIdBits;

    /**
     * 生成序列的掩码，这里为 4095 (0b111111111111 = 0xfff =4 095)
     */
    private final long sequenceMask = ~(-1L << sequenceBits);

    /**
     * 工作机器 ID(0~31)
     */
    private long workerId;

    /**
     * 数据中心 ID(0~31)
     */
    private long datacenterId;

    /**
     * 毫秒内序列(0~4095)
     */
    private long sequence = 0L;

    /**
     * 上次生成 ID 的时间截
     */
    private long lastTimestamp = -1L;

    /**
     * 构造函数
     *
     * @param workerId     工作 ID 5 位
     * @param datacenterId 数据中心 ID 5 位
     */
    private SnowFlake(long workerId, long datacenterId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        if (datacenterId > maxDataCenterId || datacenterId < 0) {
            throw new IllegalArgumentException(String.format("datacenter Id can't be greater than %d or less than 0", maxDataCenterId));
        }
        this.workerId = workerId;
        this.datacenterId = datacenterId;
    }

    /**
     * 获得下一个ID (该方法是线程安全的)
     *
     * @return SnowFlakeId
     */
    private synchronized long nextId() {
        //获取当前时间
        long timestamp = timeGen();
        /**
         * 如果当前时间小于上一次 ID 生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
         * 分布式情况下可能多个服务器不在同一时区，造成 id 重复
         */
        if (timestamp < lastTimestamp) {
            // TODO 这个时候如何补救，以及采用什么方法来解决这个问题
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }
        //如果是同一时间生成的，同一毫秒内最多生成 4096 个 id
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            // 如果一毫秒生成的 id 大于 4096（4096 一循环），则生成时间戳保证正确性
            if (sequence == 0) {
                // 阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            // 时间戳改变，毫秒内序列重置
            sequence = 0L;
        }

        // 记录上次生成 ID 的时间截
        lastTimestamp = timestamp;
        // 打印二进制的数字，便于查看
        binaryDisplay((timestamp - startTimestamp), timestampLeftShift, datacenterId, dataCenterIdShift, workerId, workerIdShift, sequence);

        // 移位并通过或运算拼到一起组成 64 位的 ID
        long result = ((timestamp - startTimestamp) << timestampLeftShift)
                | (datacenterId << dataCenterIdShift)
                | (workerId << workerIdShift)
                | sequence;
        System.out.println();
        System.out.println("最后计算结果");
        System.out.println("ID：" + result);
        return result;
    }

    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     *
     * @param lastTimestamp 上次生成ID的时间截
     * @return 当前时间戳
     */
    private long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    /**
     * 返回以毫秒为单位的当前时间
     *
     * @return 当前时间(毫秒)
     */
    private long timeGen() {
        return System.currentTimeMillis();
    }

    /**
     * 数字转二进制
     *
     * @param num 十进制数字
     */
    private void num2Binary(long num, boolean isStamp, boolean isCenter, boolean isWorker) {
        StringBuilder builder = new StringBuilder();
        int spilt = 0;
        for (int i = 63; i >= 0; i--) {
            spilt++;
            builder.append(num >>> i & 1);
            if (spilt % 8 == 0) {
                builder.append(" ");
            }
            if (isStamp) {
                if (spilt == 1 || spilt == 42) {
                    builder.append("|");
                }
            }
            if (isCenter) {
                if (spilt == 47) {
                    builder.append("|");
                }
            }
            if (isWorker) {
                if (spilt == 52) {
                    builder.append("|");
                }
            }

        }
        System.out.println(builder);

    }

    private void binaryDisplay(long stamp, long stampShift, long centerId, long centerIdShift, long workerId, long workIdShift, long sequence) {
        // 时间戳移位
        System.out.println("时间戳时间：" + stamp);
        emptyBlank((int) (stampShift + 3), "|<--左移" + stampShift + "位");
        num2Binary(stamp, true, false, false);
        num2Binary((stamp << timestampLeftShift), true, false, false);
        emptyBlank(49, "|-->后面补0");
        // dataCenterId
        System.out.println("datacenterId：" + centerId);
        emptyBlank(19, "|<--左移" + centerIdShift + "位");
        num2Binary(centerId, true, true, true);
        num2Binary((centerId << centerIdShift), true, true, true);
        emptyBlank(55, "|-->后面补0");
        // workerId
        System.out.println("workerId：" + workerId);
        emptyBlank(13, "|<--左移" + workIdShift + "位");
        num2Binary(workerId, true, true, true);
        num2Binary((workerId << workIdShift), true, true, true);
        emptyBlank(62, "|-->后面补0");
        // sequence
        System.out.println("sequence：" + sequence);
        num2Binary(sequence, false, false, false);

        //求或运算
        System.out.println();
        System.out.println("由以上4个数据进行或运算");
        num2Binary((stamp << timestampLeftShift), true, true, true);
        num2Binary((centerId << centerIdShift), true, true, true);
        num2Binary((workerId << workIdShift), true, true, true);
        num2Binary(sequence, true, true, true);
        System.out.println("OR--------------------------------------------------------------------------");
        num2Binary((stamp << timestampLeftShift)
                | (datacenterId << dataCenterIdShift)
                | (workerId << workerIdShift)
                | sequence, true, true, true);
    }

    private void emptyBlank(int start, String explain) {
        String blank = "                                                                        ";
        StringBuilder builder = new StringBuilder(blank);
        System.out.println(builder.replace(start, totalBit, explain));
    }

    public static void main(String[] args) {
        System.out.println();
        SnowFlake snowFlake = new SnowFlake(3, 5);

        snowFlake.nextId();
    }
}
```

其中一次执行结果：
``` java
时间戳时间：30918367028
                         |<--左移22位
0|0000000 00000000 00000000 00000111 00110010 11|100000 11010111 00110100 
0|0000001 11001100 10111000 00110101 11001101 00|000000 00000000 00000000 
                                                 |-->后面补0
datacenterId：5
                   |<--左移17位
0|0000000 00000000 00000000 00000000 00000000 00|00000|0 0000|0000 00000101 
0|0000000 00000000 00000000 00000000 00000000 00|00101|0 0000|0000 00000000 
                                                       |-->后面补0
workerId：3
             |<--左移12位
0|0000000 00000000 00000000 00000000 00000000 00|00000|0 0000|0000 00000011 
0|0000000 00000000 00000000 00000000 00000000 00|00000|0 0011|0000 00000000 
                                                              |-->后面补0
sequence：0
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

由以上4个数据进行或运算
0|0000001 11001100 10111000 00110101 11001101 00|00000|0 0000|0000 00000000 
0|0000000 00000000 00000000 00000000 00000000 00|00101|0 0000|0000 00000000 
0|0000000 00000000 00000000 00000000 00000000 00|00000|0 0011|0000 00000000 
0|0000000 00000000 00000000 00000000 00000000 00|00000|0 0000|0000 00000000 
OR--------------------------------------------------------------------------
0|0000001 11001100 10111000 00110101 11001101 00|00101|0 0011|0000 00000000 

最后计算结果
ID：129681030499676160
``` 

### 参考资料

[http://chuansong.me/n/2459549](http://chuansong.me/n/2459549) <br>
[https://www.jianshu.com/p/2e57acbe4a19](https://www.jianshu.com/p/2e57acbe4a19)<br>
[https://blog.csdn.net/u010372981/article/details/68924830](https://blog.csdn.net/u010372981/article/details/68924830)<br>
[https://segmentfault.com/a/1190000011282426](https://segmentfault.com/a/1190000011282426)<br>
[http://k.2dfire.net/pages/viewpage.action?pageId=295043535](http://k.2dfire.net/pages/viewpage.action?pageId=295043535)<br>
[https://tech.meituan.com/MT_Leaf.html](https://tech.meituan.com/MT_Leaf.html)
