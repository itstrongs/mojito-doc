官方文档：[https://arthas.aliyun.com/doc/](https://arthas.aliyun.com/doc/)

## 使用方法
```bash
# 下载工具
curl -O https://arthas.aliyun.com/arthas-boot.jar
# 查看运行的 java 进程信息
jps -mlvV
# 筛选 java 进程信息
jps -mlvV | grep Arthas
# 运行方式1，先运行，在选择 Java 进程 PID
java -jar arthas-boot.jar 1
# 命令格式
命令 + 类路径 方法名 + 参数
```

## 常用功能

### 查看出入参
```bash
# 出参
watch [类路径] [方法] "{returnObj}" -x 4
# 入参
watch [类路径] [方法] "{params}"
# 出入参
watch [类路径] [方法] "{params, returnObj}" -x 2
# 入参，仅当方法抛出异常时才输出
watch [类路径] [方法] {params[0], throwExp} -e
# 同时监控入参，返回值，及异常
watch [类路径] [方法] "{params, returnObj, throwExp}" -e -x 2
```

### 热部署
先修改 class 文件，再 redefine 加载 class。修改 class 有两种方式，一是通过 jad 反编译 class 为 java 文件，修改后 mc 编译成 class 文件，二是直接 IDE 修改源文件编译成 class 传到服务器上。

```bash
# 先反编译出 class 源码
jad --source-only com.example.demo.arthas.user.UserController > /tmp/UserController.java

# 然后使用外部工具编辑内容，再编译成class，如果编译不成功，可以用本地 ide 编译成 class 文件后，再上传替换
mc /tmp/UserController.java -d /tmp

# 最后，重新载入定义的类，就可以实时验证你的猜测了
redefine /tmp/com/example/demo/arthas/user/UserController.class
```

### 其他
```bash
## thread 常见用法
# 查看最繁忙的三个线程栈信息
thread -n 3
# 以直观的方式展现所有的线程情况
thread
# 找出当前阻塞其他线程的线程
thread -b

## 我新写了一个类或者一个方法，我想知道新写的代码是否被部署了
# 即可以找到需要的类全路径，如果存在的话
sc *MyServlet
# 查看这个某个类所有的方法
sm pdai.tech.servlet.TestMyServlet *
# 查看某个方法的信息，如果存在的话
sm pdai.tech.servlet.TestMyServlet testMethod

## 测试某个方法的性能问题
monitor -c 5 demo.MathGame primeFactors

1. watch：查看方法出入参
1. trace：查看方法调用链路，可以查看方法调用时间，看看哪个方法调用的慢。
1. thread：一目了然的了解系统的状态，哪些线程比较占cpu？他们到底在做什么
1. sc：查找 JVM 中已经加载的类
1. stack：查看方法的调用堆栈
1. classloader：了解当前系统中有多少类加载器，以及每个加载器加载的类数量，帮助您判断是否有类加载器泄露。
```