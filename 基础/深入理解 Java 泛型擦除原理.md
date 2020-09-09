我们都知道 Java 中的泛型可以在编译期对类型检查，避免类型强制转化带来的问题，保证代码的健壮性。不同语言对泛型的支持也不一样，Java 中的泛型类型在编译期会擦除，下面一个例子可以证明这一点：

```java
	public static void main(String[] args) throws Exception {
        List<String> list = new ArrayList<>();
        list.add("abc");
        Class<? extends List> listClass = list.getClass();
        Method addMethod = listClass.getMethod("add", Object.class);
        addMethod.invoke(list, 10);
        System.out.println(list.size());
    }
```
 
上面这个例子中定义了一个 `String` 类型的集合，通过反射获取 `add` 方法并调用该方法添加了一个 `int` 类型的元素，代码执行过程中并不会出现异常，而且能正常输出 size = 2。

因为泛型会被擦除，所以很多人都认为在代码运行期间是无法得知泛型参数类型的。最近重温 Java 基础语法的时候，[Java Reflection - Generics](http://tutorials.jenkov.com/java-reflection/generics.html) 在文章开头认为这种结论不是很准确，因为通过反射可以获取到泛型的具体类型，就像下面这样：

```java
public class GenericsSamples {
    public static List<String> getStringList() {
        return new ArrayList<>();
    }

    public static void main(String[] args) throws NoSuchMethodException {

        Method method = GenericsSamples.class.getMethod("getStringList", null);
        Type returnType = method.getGenericReturnType();
        if (returnType instanceof ParameterizedType) {
            ParameterizedType type = (ParameterizedType) returnType;
            Type[] typeArguments = type.getActualTypeArguments();
            for (Type typeArgument : typeArguments) {
                Class<?> typeArgClass = (Class<?>) typeArgument;
                System.out.println("typeArgClass = " + typeArgClass);
            }
        }
    }
}
```

`typeArgClass` 输出的类型为 `class java.lang.String`，为什么编译期间被擦除的泛型，在代码运行期间能获取到呢？原因是为了让虚拟机解析泛型类型，在虚拟机规范里引入了 `LocalVariableTypeTable` 属性，我们可以通过 `javap -v class` 反编译方式查看。

```java
public class GenericsDemo {
    public static void main(String[] args) {

        List<String> stringList = new ArrayList<>();
        List<Integer> integerList = new ArrayList<>();
        System.out.println(stringList.getClass().equals(integerList.getClass()));
    }
}
```

反编译上面 `GenericsDemo` 的 class 文件，main 方法编译信息如下：

![](https://raw.githubusercontent.com/zchen96/java-memo/master/image/基础/generics1.png)

从上面的编译信息可以看出属性 `LocalVariableTypeTable` 的 `Signature` 中保存了泛型的类型信息，因此我们可以通过反射在运行阶段获取到它。到这里可能会有人有个疑问：泛型不是被擦除了吗，为啥还能保存到 `Signature` 中，原因是它和我们所说的泛型擦除是两码事，下面再来看一个例子：

```java
public class GenericSamples<T> {

    private T param;

    public T getParam() { return param; }

    public void setParam(T param) { this.param = param; }

    public static void main(String[] args) {
        GenericSamples<Integer> integerGenericSamples = new GenericSamples<>();
        integerGenericSamples.setParam(10);
        System.out.println(integerGenericSamples.getParam());
    }
}
```

下面是 get 与 set 方法反编译的结果：

![](https://raw.githubusercontent.com/zchen96/java-memo/master/image/基础/generics2.png)

set 与 get 方法通过反编译后并不会保留类型信息，统一处理成了 `Object`，这个才是我们通常所说的泛型擦除，由于被编译器当作 `Object` 类型处理，那么我们就可以通过反射 set 任意类型的参数。

和 `LocalVariableTypeTable` 类似的还有一个 `LocalVariableTable`，不同的地方在于前者的 `Signature` 相较于后者的 `descriptor` 保存了泛型信息，关于这两个属性更多的介绍可以查看官方的介绍：[4.7.13. The LocalVariableTable Attribute](https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-4.html)。

参考：

[Java Reflection - Generics](http://tutorials.jenkov.com/java-reflection/generics.html) <br>
[深入探索Java泛型的本质 | 泛型](https://www.itzhai.com/jvm/exploring-the-nature-of-java-generics.html) <br>

