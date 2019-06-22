MySQL支持多种类型，大致将其分为三类：数值、日期/时间和字符(字符串)类型。合理地定义字段的类型对 SQL 的优化是非常重要的。

下面会介绍一些常用的数据类型，关于更多的数据类型，可以参看官方的介绍：
[https://dev.mysql.com/doc/refman/8.0/en/data-types.html](https://dev.mysql.com/doc/refman/8.0/en/data-types.html)

### 一、数值型

#### 1.1整值型
<table class="reference">
<tbody>
<tr>
<th width="15%">
类型
</th>
<th width="10%">
大小
</th>
<th width="30%">
范围（有符号）
</th>
<th width="30%">
范围（无符号）
</th>
<th width="15%">
用途
</th>
</tr>
<tr>
<td>
TINYINT
</td>
<td>
1 字节
</td>
<td>
(-128，127)
</td>
<td>
(0，255)
</td>
<td>
小整数值
</td>
</tr>
<tr>
<td>
SMALLINT
</td>
<td>
2 字节
</td>
<td>
(-32 768，32 767)
</td>
<td>
(0，65 535)
</td>
<td>
大整数值
</td>
</tr>
<tr>
<td>
MEDIUMINT
</td>
<td>
3 字节
</td>
<td>
(-8 388 608，8 388 607)
</td>
<td>
(0，16 777 215)
</td>
<td>
大整数值
</td>
</tr>
<tr>
<td>
INT 或 INTEGER
</td>
<td>
4 字节
</td>
<td>
(-2 147 483 648，2 147 483 647)
</td>
<td>
(0，4 294 967 295)
</td>
<td>
大整数值
</td>
</tr>
<tr>
<td>
BIGINT
</td>
<td>
8 字节
</td>
<td>
(-9 233 372 036 854 775 808，9 223 372 036 854 775 807)
</td>
<td>
(0，18 446 744 073 709 551 615)
</td>
<td>
极大整数值
</td>
</tr>
</tbody>
</table>

MySql 中整值数据类型默认是有符号的，也就是说支持负数。下面以 `INT` 类型为例测试一下

创建一个数据库：

```
CREATE TABLE t_int (
	num INT
);
```

插入一条数据（负数）并查看数据结果：

![这里写图片描述](https://img-blog.csdn.net/20180509161246965?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

如何使整值类型是无符号的呢？其实很简单，只要在列名后面加上 `UNSIGNED` 约束即可。下面也通过一个例子测试一下

创建一个数据表：

```
CREATE TABLE t_int2 (
	num INT UNSIGNED
);
```
插入一条数据（负数）并查看数据结果：

![这里写图片描述](https://img-blog.csdn.net/20180509161922762?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)
通过上面的图可以看出来在设置`num`字段为无符号的时候，数据表报错，不再允许插入负数数据。

通过查看数据表结构的时候还会发生一个变化，就是 `INT` 的长度由默认的 11（自己可以测试一下） 变成了 10。那么 `INT` 类型的长度有什么用呢？首先可以确定的是，这个长度与范围无关，因为范围是数据类型决定的。 `INT` 类型的长度的作用是用于 0 填充，也就是说如果查询结果值的长度小于其 `INT`的长度，前面使用 0 填充，默认是不填充的。可以使用 `ZEROFILL` 约束来设置开启 0 填充。**如果设置了 `ZEROFILL` 约束，那么这个整值数据类型变成无符号的。**

通过一个例子测试一下

创建一个数据表：
```
CREATE TABLE t_int3 (
	num INT ZEROFILL
);
```
查看数据表结构：

![这里写图片描述](https://img-blog.csdn.net/20180509164409948?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

插入一条数据（只能是有符号的）并查看数据结果：

![这里写图片描述](https://img-blog.csdn.net/20180509164304191?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

#### 1.2浮点型

<table class="reference">
<tbody>
<tr>
<th width="10%">
类型
</th>
<th width="10%">
大小
</th>
<th width="35%">
范围（有符号）
</th>
<th width="30%">
范围（无符号）
</th>
<th width="15%">
用途
</th>
</tr>
<tr>
<td>
FLOAT
</td>
<td>
4 字节
</td>
<td>
(-3.402 823 466 E+38，-1.175 494 351 E-38)，0，(1.175 494 351 E-38，3.402 823 466 351 E+38) 
</td>
<td>
0，(1.175 494 351 E-38，3.402 823 466 E+38)
</td>
<td>
单精度<br>浮点数值
</td>
</tr>
<tr>
<td>
DOUBLE
</td>
<td>
8 字节
</td>
<td>
(-1.797 693 134 862 315 7 E+308，-2.225 073 858 507 201 4 E-308)，0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308)
</td>
<td>
0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308)
</td>
<td>
双精度<br>浮点数值
</td>
</tr>
</tbody>
</table>

浮点类型数值，默认也是有符号的，同样支持负数类型的数据，这里就不作测试了。FLOAT 与 DOUBLE 的精确写法是 FLOAT(M, D) 和 DOUBLE (M, D) ，M 和 D 默认是省略的。

M：表示整数位长度加上小数位的最大长度，超会报错
D：表示小数部位允许的的长度，如果不到用 0 填充，超出部分四舍五入进位或舍位

通过一个例子说明，以 `FLOAT` 为例

创建测试表：

```python
# 允许整数位与小数位的最大长度为5，小数位长度为2
CREATE TABLE t_float (
	num FLOAT(5, 2)
);
```

插入一条数据（小数位在允许的长度内）并查看数据：

![这里写图片描述](https://img-blog.csdn.net/20180509172051182?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

插入一条数据（小数位超出允许的长度）并查看数据：

![这里写图片描述](https://img-blog.csdn.net/20180509172304292?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

插入一条数据（整数位加小数位超出允许的最大长度）并查看数据：

![这里写图片描述](https://img-blog.csdn.net/20180509172436654?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

一般我们在使用浮点类型数据的时候，不用设置 M 和 D 的值，默认的情况下 MySql 自己会根据用户插入数据的精度来决定精度。只有在一些特殊需求的情况下才需要设置 M 和 D。

#### 1.3定点型

<table class="reference">
<tbody>
<tr>
<th width="10%">
类型
</th>
<th width="10%">
大小
</th>
<th width="10%">
范围（有符号）
</th>
<th width="20%">
范围（无符号）
</th>
<th width="15%">
用途
</th>
</tr>
<tr>
<tr>
<td>
DECIMAL
</td>
<td>
对DECIMAL(M,D) ，如果M&gt;D，为M+2否则为D+2
</td>
<td>
依赖于M和D的值
</td>
<td>
依赖于M和D的值
</td>
<td>
小数值
</td>
</tr>
</tbody>
</table>

`DECIMAL` 类型的数据，M 默认为 10，D 默认为 0。其使用方法与浮点型类似，实际使用的并不多，所以这里就不作具体介绍了。

### 二、字符串型

#### 2.1字符型

<table class="reference">
<tbody>
<tr>
<th width="20%">
类型
</th>
<th width="25%">
大小
</th>
<th width="55%">
用途
</th>
</tr>
<tr>
<td>
CHAR
</td>
<td>
0-255字节
</td>
<td>
定长字符串
</td>
</tr>
<tr>
<td>
VARCHAR
</td>
<td>
0-65535 字节
</td>
<td>
变长字符串
</td>
</tr>
<tr>
<td>
TINYTEXT
</td>
<td>
0-255字符
</td>
<td>
短文本字符串
</td>
</tr>
<tr>
<td>
TEXT
</td>
<td>
0-65 535字符
</td>
<td>
长文本数据
</td>
</tr>
<tr>
<td>
MEDIUMTEXT
</td>
<td>
0-16 777 215字符
</td>
<td>
中等长度文本数据
</td>
</tr>
<tr>
<td>
LONGTEXT
</td>
<td>
0-4 294 967 295字符
</td>
<td>
极大文本数据
</td>
</tr>
</tbody>
</table>

`CHAR` 和 `VARCHAR` 类似，都可以用来存储字符类型的数据，区别是 `CHAR` 用来存储定长的字符串， 如果不指定长度，`CHAR` 类型的数据长度默认为 1， `VARCHAR`  用来存储变长的字符串，且必须要指定初始化长度。

关于定长与变长的概念，我们举例说明，假设 `CHAR`  与 `VARCHAR` 的长度都是允许存储 10 个字节，将“张三”这条数据分别插入对应的数据类型，那么“张三”在 `CHAR` 中占用全部的 10 个字节，6 个字节的数据与 4 个字节的空格填充，而在 `VARCHAR` 中只占用 6 个字节。

我们应该根据数据的特征正确的选择 `CHAR` 与 `VARCHAR` 。定长数据比如：身份证号，手机号，性别等，应该选择 `CHAR`，变长数据比如：昵称，密码等，应该选择 `VARCHAR`。

`TEXT` 对应着四种类型，这里就不作具体解释了。

还有一点需要注意的是 MySql 5.0 以上的版本， `CHAR(M)` 和  `VARCHAR(M)` 中的 M  不再是字节而是字符，但是其长度仍然不能超过对应的字节数。下面通过举例说明

创建测试表：

```
CREATE TABLE t_varchar(
	name VARCHAR(3)
);

```
插入一条数据（字节数大于3）并查看数据： 

![这里写图片描述](https://img-blog.csdn.net/20180510200817237?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

查看张三丰的字节数：

![这里写图片描述](https://img-blog.csdn.net/20180510200940550?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

“张三丰”占 9 个字节的空间大小，在设置 `VARCHAR` 大小的时候是 3， 因此`VARCHAR`  的大小是字符而不是字节。

#### 2.2二进制型

<table class="reference">
<tbody>
<tr>
<th width="20%">
类型
</th>
<th width="25%">
大小
</th>
<th width="55%">
用途
</th>
</tr>
<tr>
<td>
TINYBLOB
</td>
<td>
0-255字节
</td>
<td>
不超过 255 个字符的二进制字符串
</td>
</tr>
<tr>
<td>
BLOB
</td>
<td>
0-65 535字节
</td>
<td>
二进制形式的长文本数据
</td>
</tr>
<tr>
<td>
MEDIUMBLOB
</td>
<td>
0-16 777 215字节
</td>
<td>
二进制形式的中等长度文本数据
</td>
</tr>
<tr>
<td>
LONGBLOB
</td>
<td>
0-4 294 967 295字节
</td>
<td>
二进制形式的极大文本数据
</td>
</tr>
</tbody>
</table>

二进制类型还有 `BINARY` 和 `VARBINARY`，表用小的二进制字节串。它们两个的区别与 `CHAR` 和 `VARCHAR`类似， `BINARY` 表示定长的二进制字符串，`VARBINARY` 表示变长的二进制字符串。

`BLOB` 类型是一个二进制大对象，可以容纳可变数量的数据。可用于存储图片，文件等，但是事实上我们并不这么做，因为大二进制数据在读取的时候速度很慢。一般都是在数据库中存储这些文件的路径，通过路径获得对应的文件。

#### 2.3枚举型

枚举类型比较特殊，类似于 Java 中的枚举，要求插入的数据必须是列表中指定的值之一，并且支持英文大小写的转换，否则会报错。下面通过一个案例来说明

创建测试表：

```
CREATE TABLE t_enum(
	sex ENUM('f', 'm')
);
```

插入三条数据（列表中指定的数据）并查看数据： 

![这里写图片描述](https://img-blog.csdn.net/20180510204108859?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

插入一条数据（没有列表中指定的数据）并查看数据： 

![这里写图片描述](https://img-blog.csdn.net/20180510204237489?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NvZGVqYXM=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 三、日期型

<table class="reference">
<tbody>
<tr>
<th width="10%">
类型
</th>
<th width="10%">
大小<br>(字节)
</th>
<th width="40%">
范围
</th>
<th width="20%">
格式
</th>
<th>
用途
</th>
</tr>
<tr>
<td width="10%">
DATE
</td>
<td width="10%">
3
</td>
<td>
1000-01-01/9999-12-31
</td>
<td>
YYYY-MM-DD
</td>
<td>
日期值
</td>
</tr>
<tr>
<td width="10%">
TIME
</td>
<td width="10%">
3
</td>
<td>
-838:59:59/838:59:59
</td>
<td>
HH:MM:SS
</td>
<td>
时间值或持续时间
</td>
</tr>
<tr>
<td width="10%">
YEAR
</td>
<td width="10%">
1
</td>
<td>
1901/2155
</td>
<td>
YYYY
</td>
<td>
年份值
</td>
</tr>
<tr>
<td width="10%">
DATETIME
</td>
<td width="10%">
8
</td>
<td width="40%">
1000-01-01 00:00:00/9999-12-31 23:59:59
</td>
<td>
YYYY-MM-DD HH:MM:SS
</td>
<td>
混合日期和时间值
</td>
</tr>
<tr>
<td width="10%">
TIMESTAMP
</td>
<td width="10%">
4
</td>
<td width="40%">
1970-01-01 00:00:00/2038 -01-01 00:00:00
</td>
<td>
YYYY-MM-DD HH:MM:SS
</td>
<td>
混合日期和时间值，时间戳
</td>
</tr>
</tbody>
</table>

通过上面的图，很容易看出它们的区别。只有 `DATETIME` 与 `TIMESTAMP`都可以用于表示精确的时间，区别是字节大小不一样，并且`TIMESTAMP` 会跟随设置的时区变化而变化，而 `DATETIME` 保存的是绝对不会变化的值。
