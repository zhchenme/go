我们知道 Java 中的 `static` 关键字表示静态，可以用于修饰字段、方法、代码块、类（静态内部类）。但是除了这些作用外，还有一个就是静态导包。

静态导包是 jdk1.5 提供的一种新的机制，使用方式为 `import static packageName.className.*` 或 `import static packageName.className.methodName`，其中 `*` 表示导入类的所有静态方法。静态导包后，当调用类的静态方法时，不需要加上类名。

下面是一个静态导包的 demo

`StaticDemo` 类
```java
package com.jas.test;

public class StaticDemo {
    public static void sayHi() {
        System.out.println("Hi");
    }
    
    public static void sayBye() {
        System.out.println("Bye");
    }
}
```
静态导包测试

```java
package com.jas.test;

import static com.jas.test.StaticDemo.*;

public class StaticDemoDriven {
    public static void main(String[] args) {
        sayHi();
        sayBye();
    }
}
```
静态导包简化了 `StaticClass.staticMethod()` 调用静态方法的操作，当调用静态方法时，就像调用当前类的静态方法一样简洁。