
## Hook 线程以及捕获线程执行异常
　　在 Thread 类，关于线程运行时异常的 API 有四个：
  
- public void setUncaughtExceptionHandler（UncaughtExceptionHandler eh），为某个特定线程指定 UncaughtExceptionHandler；
- public static void setDefaultUncaughtExceptionHandler（UncaughtExceptionHandler eh），设置全局的 UncaughtExceptionHandler；
- public UncaughtExceptionHandler getUncaughtExceptionHandler()，获取特定线程的 UncaughtExceptionHandler；
- public static UncaughtExceptionHandler getUncaughtExceptionHandler()，获取全局的 UncaughtExceptionHandler。

### UncaughtExceptionHandler 介绍
　　线程在执行单元中是不允许抛出啊 check 异常，并且线程运行在自己的上下文中，派生它的线程将无法直接获得它运行中出现的异常信息。可使用 UncaughtExceptionHandler 接口，当线程在运行过程中出现异常时，会回调 UncaughtExceptionHandler 接口，从而得知哪个线程在运行时出错，以及出现什么样的错误。
  
```java
@FunctionalInterface
public interface UncaughtExceptionHandler {
    void uncaughtException(Thread t, Throwable e);
}
```

　　UncaughtExceptionHandler 是一个 FunctionalInterface，只有一个抽象方法，该回调接口会被 Thread 中的 dispatchUncaughtException 方法调用，如下：
  
```java
private void dispatchUncaughtException(Throwable e) {
    getUncaughtExceptionHandler().uncaughtException(this, e);
}
```

　　当线程在运行过程中出现异常时，JVM 会调用 dispatchUncaughtException 方法，该方法会将对应线程实例以及异常信息传递给回调接口。

### UncaughtExceptionHandler 源码分析
　　getUncaughtExceptionHandler 会先判断当前线程是否设置了 handler，如果有则执行线程自己的 uncaughtExceptionHandler，否则就到所在的 ThreadGroup 中获取，ThreadGroup 同样也实现了 UncaughtExceptionHandler 接口。

```java
public UncaughtExceptionHandler getUncaughtExceptionHandler() {
    return uncaughtExceptionHandler != null ?
        uncaughtExceptionHandler : group;
}


/**
 * ThreadGroup 类的 uncaughtException 方法
 */
public void uncaughtException(Thread t, Throwable e) {
    // 如果该 ThreadGroup 有父 ThreadGroup，则直接调用父 Group 的 uncaughtException 方法
    if (parent != null) {
        parent.uncaughtException(t, e);
    } else {
        Thread.UncaughtExceptionHandler ueh =
            Thread.getDefaultUncaughtExceptionHandler();
        // 设置了全局默认的 UncaughtExceptionHandler，则调用 uncaughtException 方法
        if (ueh != null) {
            ueh.uncaughtException(t, e);
        } else if (!(e instanceof ThreadDeath)) {
            // 如果没有父 ThreadGroup，也没设置全局默认的 UncaughtExceptionHandler，则会
            // 直接将异常的堆栈信息定位到 System.err 中
            System.err.print("Exception in thread \""
                             + t.getName() + "\" ");
            e.printStackTrace(System.err);
        }
    }
}
```

### 注入钩子线程
　　JVM 进程的退出是由于 JVM 进程中没有活跃的非守护线程，或者收到系统中断信号，向 JVM 程序注入一个 Hook 线程。在 JVM 进程退出时，Hook 线程会启动执行，通过 Runtime 可以为 JVM 注入多个 Hook 线程。<br />
　　开发中经常遇到 Hook 线程，比如为防止某个程序被重复启动，在进程启动时会创建一个 lock 文件，进程收到中断信号会删除这个 lock 文件，在 mysql 服务器、zookeeper、kafka 等系统中都能看到 lock 文件的存在。<br />
　　Hook 线程能帮助程序获得进程中的中断信号，
  
- Hook 线程只有在收到退出信号的时候会被执行，如果在 kill 的时候使用参数 -9，Hook 线程不会得到执行，进程会立即退出；
- Hook 线程也可执行一些资源释放的工作，比如关闭文件句柄、socket 连接、数据库 Connection 等；
- 尽量不要在 Hook 线程执行一些耗时非常长的操作，会导致程序迟迟不能退出。
