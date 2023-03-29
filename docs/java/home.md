# Java 语法篇

<!--http://cdn.liufq.com/Fq2r8SehqPbAW2m-N--k2C-pBhZt-->

![](http://cdn.liufq.com/FmUJ2oB1r3GaQ37Mt8-uGN0jDl0G ':size=75%')

## 为什么重写 equals 的时候需要重写 hashCode ？

**hashCode()**：是一个 native 方法，返回的是对象的 hash 值，同一个对象 hash 值一定相同。

**equals()**：没有重写 equals 时等同于 ==，对于基本数据类型，比较的是对象的值，对于引用类型，比较的是对象的地址。

重写 equals 主要目的是为了让我们认为相等的不同对象识别为相同对象，相同对象应该有相同的 hashCode，没有重写 hashCode 的话这两个对象就不会有相同的 hashCode，在一些使用 hashCode 判断相同对象的场景会产生问题，比如 HashMap。

## String、StringBuilder 和 StringBuffer 有什么区别？
1. **可变性**：String 是 final 修饰不可变的，每次对 String 的操作都会生成一个新 String 对象，而 StringBuffer 和 StringBuilder 没有 final 修饰，是可变的。所以对于频繁修改的对象应该避免用 String；

1. **线程安全**：String 是 final 修饰不可变的，所以天然是线程安全的。StringBuffer 有 synchronized 同步，是线程安全的，StringBuilder 没有，不是线程安全的。所以 StringBuilder 性能会好一点，StringBuffer 适合有线程安全问题场景。

## StringBuild、StringBuffer 源码分析

使用 char[] 存储字符串，初始容量为字符串长度 + 16；count 存储字符串长度。

**扩容机制**：ensureCapacityInternal(count + len)，当存储的字符串长度 + 追加的字符串长度 > 数组长度时发生扩容，使用 Arrays.copyOf(char[] original, int newLength) 实现扩容，扩容大小为：append 字符串的大小和 char 数组的大小 * 2 + 2 取最大值。

**数据存储**：通过 String 的 getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) 方法，将要添加的 String 复制到 char 数组，从 count 位置开始存放，native 实现。

**相同点**：都继承了 AbstractStringBuilder，共用了很多代码。

**区别**：StringBuffer append 方法加了 synchronized，toString 缓存了字符串到 toStringCache。

## Integer a1 = 10, a2 = 10，== 的结果？
结果是 true。== 比较的是两个对象的地址，同一个对象才会返回 true，但是 Integer 内部做了处理，通过 static 代码块缓存了 [-128,127] 区间的数据，当值在这个区间时不会重新创建对象，直接从缓存里取，所以是同一个对象。超过这个区间是 false。

## 什么是 Java 反射？

Java 反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为 Java 语言的反射机制。

## Java 有哪些移位运算符？
* <<：左移运算符，x << 1，相当于 x 乘以 2 (不溢出的情况下)，低位补 0
* \>>：带符号右移，x >> 1，相当于 x 除以 2，正数高位补 0，负数高位补 1
* \>>>：无符号右移，忽略符号位，空位都以 0 补齐

## 什么是 SPI 机制？
SPI（Service Provider Interface）是 JDK 内置的一种**服务提供发现机制**，可以用来**启用框架扩展和替换组件**。Java 中 SPI 机制主要思想是**将装配的控制权移到程序之外**，在模块化设计中这个机制尤其重要，其核心思想就是**解耦**。

**实现方式**：
1. 服务的提供者提供了一种接口的实现
1. 在 classpath 下的 META-INF/services/ 目录里创建一个以服务接口命名的文件，这个文件里的内容就是这个接口的具体的实现类
1. 当其他的程序需要这个服务的时候，就可以通过查找这个 jar 包的 META-INF/services/ 中的配置文件，配置文件中有接口的具体实现类名，可以根据这个类名进行加载实例化，就可以使用该服务了

## 什么是 Java NIO？
NIO 主要有三大核心部分：Channel（通道）、Buffer（缓冲区）、Selector（选择区）。传统 IO 基于字节流和字符流进行操作，而 NIO 基于 Channel 和 Buffer 进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。Selector 用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个线程可以监听多个数据通道。

NIO 和传统 IO 之间第一个最大的区别是，IO 是面向流的，NIO 是面向缓冲区的。

## Java 浮点型

使用科学计数法标识，分为符号、有效数字、指数。

float（单精度）：4字节，32 位，1 位符号位 + 8 位指数组成（阶码位） + 23 位有效数字（尾数位）。

其中**阶码**存的是指数的移码，取值范围：[-126, 127] ≈ 1.7 * 10^38；

**尾数**存的是原码，最大值为 1.111...(23 个 1)，无限接近于 2；

所以单精度浮点数的最大值为：2 * 1.7 * 10^38 ≈ 3.4e38

> **移码**：[x]移 = X + 2 的 (n - 1) 次幂，指一个值用平移一个偏移量后的值表示，n 是这个值得位数，比如 8 位的取值范围是 [-128, 127]，移码后是 [0, 255]，这样就可以避免负数运算。<br><br>
而由于移码 0 和 255 在浮点数里有特殊用处，移码范围成了 [1, 254]，此时仍偏移 128 的话，指数范围是 [-127, 126]，为了尽量增大表示范围，浮点数的偏移量定为正常偏移量 -1，即 127，此时指数范围就是 [-126, 127]。