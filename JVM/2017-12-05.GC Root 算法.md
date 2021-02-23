在进入主题之前，我们要先知道运行时数据区域都是有哪些块内存需要进行垃圾回收。

程序计数器、虚拟机栈、本地方法栈、3 个区域都是随着线程生而生，随着线程灭而灭的；栈中的栈帧随着方法的进入和退出有条不紊的执行着进栈和出栈的操作。每一个栈帧中分配多大的内存基基本上在类结构确定下来后就已知了，因此这几个区域的内存分配和回收都具备确定性，在这几个区域就不需要过多的考虑垃圾回收的问题，因为 方法结束或者线程结束时，内存自然就跟着回收了。

而 Java 堆和方法区不一样，一个接口中方的多个实现类需要的内存可能不一样，一个方法的多个分支需要的内存可能有也不一样，我们只有在程序处于运行期间才知道会创建哪些对象，这部分内存的分配和回收都是动态的，垃圾回收机制所关注的也是这部分内存。

在 Java 堆里几乎放着所 有 Java 对象的实例，垃圾收集器在对堆进行回收之前，第一件事就是要确定哪些对象是存活的，哪些对象已经"死去"。

### 一、引用计数

在进行 GC  Root 之前，我们先来了解一个判断对象是否存活的算法：引用计数法。该算法的是这样的：给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加 1 ；当引用失效时，计数器值就减 1 ；任何时刻该计数器的值为 0 时，就说明该对象是不可能再被使用的。

引用计数算法 (Reference Counting) 的实现很简单，效率也很高，在大部分情况下都是一个不错的算法。但是目前 Hot Spot 虚拟机并没有使用这种算法来管理内存，因为这种算法很难解决对象之间相互引用的问题

。下面就是一个简单的例子，创建了两个对象，这两个对象都相互引用着对方，导致他们的引用计数器都不为 0，所以引用计数法就不能通知 GC 收集器收集他们，但这两个对象除了相互引用之外没有任何其他的用途。我打印了 GC 日志以提供分析，为了防止大对象被直接分配到老年代，我把大对象分配到老年代的内存设置成了 3m 大小。

```
public class ReferenceCountingGC {
    public Object instance = null;
    private static final int _1MB = 1024*1024;
    private byte[] bigSize = new byte[2 * _1MB];

    public static void main(String[] args) {
        ReferenceCountingGC obj1 = new ReferenceCountingGC();
        ReferenceCountingGC obj2 = new ReferenceCountingGC();
        
        obj1.instance = obj2;
        obj2.instance = obj1;

        System.gc();
    }
}

```
![这里写图片描述 ](http://img.blog.csdn.net/20171129111332204?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZWphcw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

由上面的图 (如果上图看不清楚你可右键在新标签页面查看图片) 可以看出在进行垃圾回首之前新生代被占用了 4744 K 大小，在 GC 之后新生代的内存就被释放了。因此这也可以说明虚拟机并不是通过引用计数法判断对象是否能存活的。


### 二、GC Root 可达性分析

目前的主流程序语言的实现中，都是通过可达性分析 (Reachability Analysis) 来判定对象是否存活的。这个算法的基本思路就是一系列的称为"GC Roots" 的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径被称为引用链 (Reference Chain)，当一个对象到 GC Roots 没有任何引用链相连时，则证明这个对象是不可用的 (图片来自百度搜索)。如图所示，object5、object6、object7 虽然互有关联，但是它们到 GC Roots 是不可达的，所以它们将被判定为可回收的对象。

![这里写图片描述 ](http://img.blog.csdn.net/20171129115552919?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvY29kZWphcw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

在 Java 语言中可以作为 GC Roots 的对象包括下面这几种：

 - 虚拟机栈中引用的对象
 - 本地方法栈中引用的对象
 - 堆中类静态属性引用的对象

### 参考

《深入理解Java 虚拟机》周志明 著