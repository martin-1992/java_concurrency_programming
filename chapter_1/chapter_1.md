
## 快速认识线程
　　对计算机来说，每一个任务就是一个进程，在每一个进程内部至少要有一个线程在运行。每一个线程都有自己的局部变量表、程序计数器（指向正在执行的指令指针）以及各自的生命周期。
  
### 线程的生命周期详解
　　线程的生命周期可分为 5 个主要阶段：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_1/chapter_1_p1.png)
  
- **线程的 NEW 状态。**使用关键字 new 创建一个 Thread 对象时，此时它并不处于执行状态，因为在没有 start 之前，该线程并不存在，与用 new 创建普通 Java 对象没什么区别；
- **线程的 RUNNABLE 状态。**进入该状态需调用 start 方法，此时具备线程执行的资格，但线程并没有真正执行，而是在等待 CPU 的调度；
- **线程的 RUNNING 状态。**一旦 CPU 通过轮询或者其他方法从任务可执行队列中选中了该线程，此时才真正执行逻辑代码。在该状态中，有几种状态转换：
    1. 调用 stop 方法（不推荐）或判断某个逻辑标识，进入 TERMINATED 状态；
    2. 调用 sleep 或 wait 方法，加入 waitSet 中，进入 BLOCKED 状态；
    3. 进行某个阻塞的 IO 操作，比如因网络数据的读写，进入了 BLOCKED 状态；
    4. 获取某个锁资源，从而加入到该锁的阻塞队列中，进入 BLOCKED 状态；
    5. 由于 CPU 的调度器轮询，使该线程放弃执行，进入 RUNNABLE 状态；
    6. 线程主动调用 yield 方法，放弃 CPU 执行权，进入 RUNNABLE 状态。
- **线程的 BLOCKED 状态。**前面已提到进入 BLOCKED 状态的原因，该状态的有几种状态切换：
    1. 调用 stop 方法（不推荐）或意外死亡（JVM Crash），进入 TERMINATED 状态；
    2. 线程阻塞的操作结束，进入到 RUNNABLE 状态；
    3. 线程完成了指定时间的休眠，进入到 RUNNABLE 状态；
    4. Wait 中的线程被其他线程 notify / notifyall 唤醒，进入 RUNNABLE 状态；
    5. 线程获取到了某个锁资源，进入 RUNNABLE 状态；
    6. 线程在阻塞过程中被打断，比如其他线程调用了 interrupt 方法，进入 RUNNABLE 状态。
- **线程的 TERMINATED 状态**。TERMINATED 是一个线程的最终状态，在该状态中线程将不会切换到其它任何状态，表示该线程的整个生命周期都结束了，如下情况将进入该状态：
    1. 线程运行正常结束；
    2. 线程运行出错意外结束；
    3. JVM Crash，导致所有线程都结束。

### 线程的 start 方法剖析
　　下面为 Thread start 方法的源码：
  
  
```java
public synchronized void start() {
    // 第二次调用线程，即不为 0，将会抛出 IllegalThreadStateException 异常
    if (threadStatus != 0) {
        throw new IllegalThreadStateException();
    }
    // 线程启动后，会加入到一个 ThreadGroup 中
    group.add(this);
    
    boolean started = false;
    
    try {
        // 本地方法，会调用 run 方法
        start0();
        started = true;
    } finally {
        try {
            if (!started) {
                group.threadStartFailed(this);
            }
        } catch (Throwable ignore) {
        }
    }
}
```

　　最核心的部分是 start0 这个本地方法，也是 JNI 方法，而 run 方法是被 JNI 方法 start0 调用的。
  
```java
private native void start0();
```

#### 模板设计模式在 Thread 中的应用
　　线程的真正执行逻辑在 run 方法中，通常把 run 方法称为线程的执行单元，重写 run 方法，用 start 方法启动线程。<br />
　　Thread 的 run 和 start 是一个典型的模板设计模式，父类编写算法结构代码，子类实现逻辑细节 run，即线程的启动步骤都由父类实现好了，按照顺序启动，只需实现步骤 run 方法的细节即可。

### Runnable 接口
　　Runnable 接口定义一个无参数无返回值的 run 方法
  
```java
public interface Runnalbe {
    void run();
}
```

　　注意，**创建线程只有一种方式就是构造 Thread 类**，而实现线程的执行单元有两种方式：
  
- 重写 Thread 的 run 方法；
- 实现 Runnable 接口的 run 方法，并且将 Runnalbe 实例用作构造 Thread 的参数。

　　下面代码是 Thread run 方法的源码，创建线程的唯一方法是构造 Thread 类，然后判断是否使用 Runnable 接口，没则调用 Thread 类的 run 方法。

```java
@Override
public void run() {
    // 如果构造 Thread 时传递了 Runnable，则会执行 Runnable 的 run 方法
    if (target != null) {
        target.run();
    }
    
    // 否则需要重写 Thread 类的 run 方法
}
```

#### 策略模式在 Thread 中的应用
　　无论是 Runnable 的 run 方法，还是 Thread 类本身的 run 方法（事实上 Thread 类也是实现 Runnable 接口）都是想将线程的控制本身和业务逻辑的运行分离开来。<br />
　　将会变化的部分抽取出来，交由子类实现，比如 Runnable 接口的 run 方法，由我们自己实现，其他不会变化的部分，比如启动、结束线程这些，则由超类定义好。<br />
　　重写 Thread 类的 run 方法和实现 Runnable 接口的 run 方法一个重要不同是，Thread 类的 run 方法是不能共享的，也就是说 A 线程不能把 B 线程的 run 方法当作自己的执行单元，而使用 Runnable 接口则可以，即使用同一个 Runnable 的实例构造不同的 Thread 实例。
