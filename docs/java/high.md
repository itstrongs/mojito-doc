## Java 常用命令
### JDK 工具
```
# 查看本地正在运行的java进程和进程ID
jps

# 查看指定 pid 的 JVM 信息
# -flags：查询 JVM 运行参数
jinfo [pid]
```

## jdk8 新特性
### 函数式编程和 lambda 表达式

面向对象编程是对数据进行抽象，函数式编程是对行为进行抽象。lambda 表达式是函数式接口的写法。

#### @FunctionalInterface【函数接口】
用作接口注解，接口必须只有一个抽象方法，当接口满足函数接口要求时，没加 @FunctionalInterface 注解的接口也会被编译器识别为函数接口。可以用 lambda 表达式调用。

```java
/**
 * 自定义函数接口
 * /
@Test
void functionalInterface() {
    IMyInterface myInterface = () -> System.out.println("自定义函数接口");
    myInterface.study();
}

@FunctionalInterface
public interface IMyInterface {
    void study();
}
```

#### 内置的四大函数接口
1. 消费型接口: Consumer< T> void accept(T t) 有参数，无返回值的抽象方法
1. 供给型接口: Supplier < T> T get() 无参有返回值的抽象方法；
1. 断定型接口: Predicate<T> boolean test(T t) 有参，但是返回值类型是固定的 boolean
1. 函数型接口: Function<T,R> R apply(T t) 有参有返回值的抽象方法；

### 其他新特性

1. **Stream 流**：a. 提高代码简洁性和可读性；b. 性能：简单迭代性能低于for循环；复杂操作优于手动循环操作；并行在多核情况下优于串行；

1. **Optional 类**：解决判空问题

1. **新的日期 API**：解决了之前日期类非线程安全、设计很差、时区处理麻烦等问题

1. 接口中可以添加使用 **default** 或者 static 修饰的方法

1. **类型注解**：

   类型注解被用来支持在 Java 的程序中做强类型检查。配合插件式的 check framework，可以在编译的时候检测出 runtime error，以提高代码质量。

   java 8 之前，注解只能是在声明的地方所使用，比如类，方法，属性；java 8里面，注解可以应用在任何地方。编译成 class 文件的时候并不包含类型注解。

> jdk8 之后每 3 年发布一个 TLS（长期维护版本）。意味着 jDK 8、11、14、17 才可能被大规模使用。

## jdk9 新特性

1. 允许接口定义私有方法
1. 默认垃圾收集器改为 G1
1. 引入 JShell 工具，让 Java 也可以像脚本语言一样来运行

## jdk10 新特性
1. 优化 G1 收集器：并行全垃圾回收器
1. 局部变量类型推断 var

## jdk11 新特性
1. 增加新的垃圾收集器 ZGC
1. 字符串加强

## jdk12 新特性
1. Switch Expressions
1. 新增了一个名为 Shenandoah 的 GC 算法，通过与正在运行的 Java 线程同时进行 evacuation 工作来减少 GC 暂停时间

## jdk13 新特性
1. 文本块升级

## jdk14 新特性
1. 删除 cms 垃圾收集器、ParallelScavenge + SerialOld GC 的垃圾回收算法组合、将 zgc 垃圾回收器移植到 macOS 和 windows 平台
1. switch 表达式成稳定版本正式可用

## 参考
* [jdk11、12、13、14面向开发者的新特性](https://blog.csdn.net/m0_38001814/article/details/88831037)