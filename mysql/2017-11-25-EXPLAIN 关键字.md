
### EXPLAIN 关键字详解

使用 EXPLAIN 关键字可以模拟优化器执行 SQL 查询语句，从而知道 MySQL 数据库是如何处理你的 SQL 语句的。因此我们可以使用该关键字知道我们编写的 SQL 语句是否是高效的，从而可以提高我们程序猿编写 SQL 的能力。

使用 EXPLAIN 关键字可以让我们知道表的读取顺序、数据读写操作的操作类型、哪些索引是可以使用的、哪些索引是实际使用的、表之间的引用、每张表有哪些行被 MySQL 优化器查询。其基本的使用语法是 EXPLAIN + SQL 语句。

在这里我使用 EXPLAIN 关键字执行了一条 SQL 语句

```
EXPLAIN SELECT * FROM t_dept WHERE dept_id = (SELECT dept_id FROM t_emp WHERE emp_id=177); 
```

下面是执行该语句所输出的结果：

![这里写图片描述 ](http://img.blog.csdn.net/20171121151105973?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMzY2MDg5NTU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

我们可以看出查询结果包含了很多字段，下面我们就来了解一下这些字段都是什么意思。

### EXPLAIN 字段详解

**id**:SELECT 查询的序列号，表示查询中执行的 SELECT 子句或操作表的顺序，id 表现的形式有三种：
1]：id 相同，执行顺序由上至下。

2]：id 不同，如果是子查询，id 的序号会递增，id 的值越大优先级越高，越先被执行。

3]：id 既有相同的又有不同的，即 id 值越大的优先被执行，如果存在 id 相同的，则在上面的执行顺序高于在下面的。

所以 MySQL 对于我们编写的 SQL 语句并不是按照我们编写的顺序执行，这一点与我们的四则运算顺序相类似，并不是在最前面的运算先执行，要考虑运算符与括号的问题。

**select_type**:首先 select_type 有好几种，表示查询的类型，主要用于区别普通查询，联合查询，子查询等复杂查询。下面逐一介绍：

<table>
  <tr >
    <th  width=10%, bgcolor=yellow >select_type</th>
    <th  width=40%, bgcolor=yellow> 介绍 </th>
  </tr>
  <tr>
    <td bgcolor=#eeeeee> SIMPLE </td>
    <td> 简单的 SELECT 查询，查询中不包括子查询或者 UNION  </td>
  </tr>
  <tr>
    <td bgcolor=#eeeeee>PRIMARY </td>
    <td> 查询中若包含任何复杂的子查询，最外层的查询会被标记为 PRIMARY </td>
  <tr>
    <td bgcolor=#eeeeee>SUBQUERY </td>
    <td> 在 SELECT 或 WHERE 列表中包含的子查询部分 (可以参照上面查询的结果) </td>
  </tr>
  <tr>
    <td bgcolor=#eeeeee>DERIVED</td>
    <td> 在 FROM 列表中包含的子查询被标记为 DERIVED (衍生)，MySQL 会递归执行这些子查询，把结果放在临时表里 </td>
  </tr>
    <tr>
    <td bgcolor=#eeeeee>UNION </td>
    <td> 若第二个 SELECT 出现在 UNION 之后，就会被标记为 UNION，若 UNION 包含在 FROM 字句的子查询中，外层的 SELECT 将被标记为：DERIVED</td>
  </tr>
    <tr>
    <td bgcolor=#eeeeee>UNION RESULT </td>
    <td> 从 UNION 表中获取结果的 SELECT</td>
  </tr>
  </table>
   <br>
   
**table**:表示这一行数据关联的数据表。

**partitions**:表示给定表所使用的分区。

**type**:表示查询语句的连接类型。type 也有很多种，下面做具体介绍：

<table>
  <tr >
    <th  width=10%, bgcolor=yellow >type</th>
    <th  width=40%, bgcolor=yellow> 介绍 </th>
  </tr>
  <tr>
    <td bgcolor=#eeeeee> system</td>
    <td> 表只有一行 (相当于系统表)，这是 const 类型的特例，一般很少出现 </td>
  </tr>
  <tr>
    <td bgcolor=#eeeeee>const  </td>
    <td> 表示通过索引一次就可以查询到数据，用于比较 primary key 或者 unique 索引。如果将主键置于 WHERE 条件中，MySQL 就将该查询转换位一个常量 </td>
  <tr>
    <td bgcolor=#eeeeee>eq_ref</td>
    <td> 唯一性索引扫描，对于每个索引键，表中只有一条记录相匹配，常用于主键或唯一索引扫描 </td>
  </tr>
  <tr>
    <td bgcolor=#eeeeee>ref </td>
    <td> 非唯一性索引扫描，返回匹配某个单独值的所有行，本质上也是一种索引访问，返回的是所有匹配某个单独之的行，属于查找和扫描的混合体 </td>
  </tr>
    <tr>
    <td bgcolor=#eeeeee>range</td>
    <td> 只检索给定范围的行，使用一个索引来选择行。比如在 WHERE 语句中出现的 BETWEEN 、IN 等的查询。这种索引的扫描范围比全表扫描要好 </td>
  </tr>
    <tr>
    <td bgcolor=#eeeeee>index </td>
    <td> 所有索引扫描，index 与 all 的最大区别是 index 类型只扫描索引树。这通常比 all 块，因为索引文件通常比数据文件要小 </td>
  </tr>
  <tr>
    <td bgcolor=#eeeeee>all</td>
    <td> 遍历全表找到匹配的行 (是从磁盘中读取数据)</td>
  </tr>
  </table>
  <br>type 的类型不止上面的几种，这里只列出了比较常见的几种类型，  根据上面的介绍我们总结出从出现的最好到最差的顺序是：system > const > eq_ref > ref > range > index > all ，但是对于一般来说，能够保证达到 range 级别就可以了，最好达到 ref 级别。

**possible_keys**:  表示在这张表可能会使用的索引，一个或多个，只是可能会使用，但是不表示一定被实际使用。

**key**:实际上使用的索引。可以为 null 表示没有使用索引。
 
**key_len**:表示索引中的字节数，可以通过该列计算查询中使用的索引的长度。在不失精度的情况下，长度越短越好。key_len 显示的值为索引字段的最大可能长度，并非实际长度，是根据表定义计算得出。

**ref**：显示索引的哪一列被使用了，一般是一个常数。

**rows**:根据表统计信息及选取索引的情况，大致估算出查到记录所需要读取的行数。

**filtered**:filtered 列给出了一个百分比的值，这个百分比值和 rows 列的值一起使用，可以估计出那些将要和 QEP 中的前一个表进行连接的行的数目。

**extra**:包含不适合在其他列中显示但是也十分重要的额外信息：

<table>
  <tr >
    <th  width=10%, bgcolor=yellow >type</th>
    <th  width=40%, bgcolor=yellow> 介绍 </th>
  </tr>
  <tr>
    <td bgcolor=#eeeeee>  Using filesort</td>
    <td>  说明 MySQL 会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。MySQL 无法利用索引完成的排序操作被称为“文件排序”。这种情况是极其糟糕的 </td>
  </tr>
  <tr>
    <td bgcolor=#eeeeee> Using temporary</td>
    <td> 使用了临时表保存中间结果，MySQL 在对查询结果排序时使用临时表， 主要是 order by 和分组查询。这种情况的出现也是极其糟糕的 </td>
  <tr>
    <td bgcolor=#eeeeee> Using  index</td>
    <td> 表示相应的查询字段覆盖了索引，避免了访问表的数据行，效率很高。</td>
  </tr>
  <tr>
    <td bgcolor=#eeeeee>Using where</td>
    <td> 表示使用了 WHERE 进行过滤 </td>
  </tr>
    <tr>
    <td bgcolor=#eeeeee>Using join where</td>
    <td> 表示使用了连接缓存 </td>
  </tr>
    <tr>
    <td bgcolor=#eeeeee>impossible where</td>
    <td> 表示 WHERE 子句的值永远为 false ，不能用来获取数据 </td>
  </tr>
  <tr>
    <td bgcolor=#eeeeee>distinct</td>
    <td> 优化 distinct 操作，表示在找到第一匹配的元组后就停止找同样的值 </td>
  </tr>
  </table>
  <br>