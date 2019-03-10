
## 深入理解 Thread 构造函数
　　Thread 构造函数，如下：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_2/chapter_2_p1.png)

### 线程的命名
　　如果没有为线程显示指定一个名字，线程会以“Thread-”作为前缀与一个自增数字进行组合，这个自增数字在整个 JVM 进程中将会不断自增，比如 Thread-0，Thread-1，Thread-2 ... 等等。<br />
　　源码如下：
 
```java
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}

private static int threadInitNumber;
private static synchronized int nextThreadNum() {
    return threadInitNumber++;
}
```
　　建议在构造 Thread 的时候，为线程赋予一个名字。

### 线程的父子关系
　　Thread 的所有构造函数，最终都会调用一个静态方法 init，源码如下：
  
```java
private void init(ThreadGroup g, Runnable target, String name, long stackSize, AccessControlContext acc) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }
    this.name = name.toCharArray();
    // 获取当前线程作为父线程
    Thread parent = currentThread();
    SecurityManager security = System.getSecurityManager();
     ...
}
```

　　线程最初状态为 new，在没有执行 start 方法前，只能算是一个 Thread 的实例，而 currentThread 用于创建它的那个线程：
  
- 一个线程的创建肯定是由另一个线程完成的；
- 被创建线程的父线程是创建它的线程。

　　这里的父线程为 main 线程，由它创建所有线程。

### Thread 与 ThreadGroup
　　在 Thread 的构造函数中，可显示指定线程的 Group，即 ThreadGroup。
    
```java
SecurityManager security = System.getSecurityManager();

if (g == null) {
    // 获取 ThreadGroup，需要显示指定
    if (security != null) {
        g = security.getThreadGroup();
    }
    
    // 如果没有显示指定一个 ThreadGroup，那么子线程将会加入父线程所在的线程组
    if (g == null) {
        g = parent.getThreadGroup();
    }
}
```
　　main 线程所在的 ThreadGroup 称为 main，如果没有显示指定 ThreadGroup，那么它将会和父线程（main 线程）同属一个 ThreadGroup。

### Thread 与 JVM 虚拟机栈
　　Thread 的构造函数中，有一个特殊的参数 stackSize。
  
```java
Thread(ThreadGroup group, Runnable target, String name, long stackSize)
```

　　一般情况下，创建线程时不会手动指定栈内存的地址空间字节数组，统一通过 xss 参数进行设置，stacksize 越大则代表着正在线程内方法调用递归的深度就越深，反之，则递归深度越浅。

### JVM 内存结构
　　JVM 在执行 Java 程序时会把对应的物理内存划分成不同的内存区域，如下：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_2/chapter_2_p2.png)

- 程序计数器，用于存放当前线程接下来要执行的字节码指令、分支、循环、跳转、异常处理等信息。在任何时候，一个处理器只执行其中一个线程的指令，为了能够在 CPU 时间片轮转切换上下文之后顺利回到正确的执行位置，每条线程都需要具有一个独立的程序计数器，此块内存区域为线程私有的；
- Java 虚拟机栈，也是线程私有的，即每个线程会有一个虚拟机栈。在线程中，方法在执行的时候会创建一个名为栈帧的数据结构，用于存放局部变量表、操作栈、动态链接、方法出口等信息，方法的调用对应着栈帧在虚拟机栈中的压栈和弹栈过程，栈内存的大小决定在一个 JVM 进程中可创建多少个线程；

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_2/chapter_2_p3.png)

- 本地方法栈。Java 中提供了调用本地方法的接口（Java Native Interface，JNI），也就是 C/C++ 程序，也是线程私有的内存区域；
- 堆内存。是 JVM 中最大的一块内存区域，被所有线程共享，Java 在运行期间创建的所有对象几乎都存放在该内存区域，又叫“GC堆”，因为是垃圾回收器重点关注的的区域；
- 方法区。被多个线程共享的内存区域，用于存储已经被虚拟机加载的类信息、常量、静态变量、即时编译器（JIT）编译后的代码等数据；
- Java 8 元空间。取代了之前版本的持久代内存区域，元空间同样是堆内存的一部分，JVM 为每个类加载器分配一块内存列表，进行线性分配，块的大小取决于类加载器的类型。

#### Thread 与虚拟机栈
　　程序计数器是比较小的内存，不会出现内存溢出，因为它是记录线程要执行的下一条指令。<br />
　　与线程创建、运行、销毁等关系比较大的是虚拟机内存，栈内存的划分决定一个 JVM 进程中可创建多少个线程。堆内存不变的情况下，栈内存越大，可创建的线程数量越小。因为虚拟机栈内存是线程私有的，每个线程一个栈内存，栈内存与线程数量成反比。<br />
　　计算线程数量的公式：线程数量 = （最大地址空间 MaxProcessMemory - JVM 堆内存 - ReservedOsMemory（系统保留内存，为 136 MB）） / ThreadStackSize（XSS，栈内存）。在 Linux 中，下面三个内核配置信息也可决定线程数量的大小：
  
- /proc/sys/kernel/threads-max；
- /proc/sys/kernel/pid_max；
- /proc/sys/vm/max_map_count；

### 守护线程
　　守护线程具备自动结束生命周期的特性，而非守护线程则不具备这个特点。<br />
　　守护线程经常用于执行一些后台任务，也称为后台线程，可关闭线程。JVM 的进程 main 线程为非守护线程，线程结束 JVM 进程也不会退出。<br />
　　调用 setDaemon 方法可设置守护线程。线程是否为守护线程和他的父线程有很大关系，如果父线程是正常线程，子线程也是，反之亦然。另外，setDaemon 方法只在线程启动前生效，如果线程死亡，设置 setDaemon 会抛出 IllegalThreadStateException 异常。

### 总结：

- 一个线程的创建肯定是由另一个线程完成的，最初的父线程为 main 线程，由它来创建其它子线程；
- 如果没有显示指定一个 ThreadGroup，那么子线程将会加入父线程所在的线程组；
- 在 JVM 内存结构中，程序计数器、本地方法栈和 Java 虚拟机栈为线程私有，堆内存和方法区为线程公有，堆内存主要存放 Java 运行期间的对象，所以是 gc 重点关注的区域；
- 守护线程具备自动结束生命周期的特性，即当线程运行完后，会自动关闭线程。
