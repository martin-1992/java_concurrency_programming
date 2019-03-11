
## Thread API 详细介绍

### 线程 sleep
　　线程 sleep 是一个静态方法，使当前线程进入指定毫秒数的休眠，暂停执行，重要一点是不会放弃 monitor 锁的所有权。<br />
　　推荐使用 TimeUnit，是对 sleep 方法提供很好的封装，使用它不用计算是时间单位的换算。如下，线程想休眠 3 小时 24 分 17 秒：
  
```java
Thread.sleep(12257000L);
TimeUnit.HOURS.sleep(3);
TimeUnit.MINUTES.sleep(24);
TimeUnit.SECONDS.sleep(17);
```

### 线程 yield
　　yield 方法属于一种启发式的方法，提醒调度器放弃当前的 CPU 资源，调用 yield 方法会使当前线程从 RUNNING 状态切换到 RUNNABLE 状态，yield 和 sleep 的区别：
  
- sleep 会导致当前线程暂停指定的时间，没有 CPU 时间片的消耗；
- yield 只是对 CPU 调度器的一个提示，如果 CPU 调度器没有忽略这个提示，会导致线程上下文的切换；
- sleep 会使线程短暂 block，并在给定时间内释放 CPU 资源；
- yield 会使 RUNNING 状态的 Thread 进入到 RUNNABLE 状态（如果 CPU 调度器没有忽略这个提示）；
- sleep 几乎百分百完成给定时间的休眠，而 yield 的提示不一定能担保；
- 一个线程 sleep，另一个线程调用 interrupt 会捕获到中断信号，而 yield 则不会。

### 设置线程优先级
　　设置线程优先级同样是一个 hint 操作，如果 CPU 比较忙，设置优先级可能会获得更多的 CPU 时间片，但是在闲时，优先级高低不会有作用，所以不要让业务严重依赖线程优先级。
  

```java
public final void setPriority(int newPriority) {
    ThreadGroup g
    checkAccess();
    // 线程优先级在 1 到 10 之间，不能超过这个区间
    if (newPriority > MAX_PRIORITY || newPriority < MIN_PRIORITY) {
        throw new IllegalArgumentException;
    }
    
    // 获取线程组，线程的最大优先级为线程组的最大优先级
    if ((g = getThreadGroup) != null) {
        if (newPriority > g.getMaxPriority() {
            newPriority = g.getMaxPrioriy();
        }
        // 设置线程优先级
        setPriority0(priority = newPriority);
    }
}
```

　　线程默认的优先级和它的父类保持一致，因为线程由父线程 main 线程创建，main 线程优先级为 5，所以派生的线程优先级为 5。

### 获取线程 ID
　　public long getId()，获取线程的唯一 ID，线程 ID 在整个 JVM 进程中是唯一的，并且从 0 开始递增。
    
### 获取当前线程
　　public static Thread currentThread()，返回当前执行线程的应用。
    
### 设置线程上下文类加载器

- public ClassLoader getContextClassLoader()，获取线程上下文的类加载器，即这个线程由哪个类加载器加载的，在没有修改上下文类加载器的情况下，保持与父线程同样的类加载器；
- public void setContextClassLoader(ClassLoader cl)，设置该线程的类加载器，打破 Java 类加载器的父委托机制。

### 线程 interrupt

- interrupt，当线程进入阻塞状态时，调用当前线程的 interrupt 方法，可打断当前线程的阻塞状态。一旦线程在阻塞情况下被打断，都会抛出一个 InterruptException 异常。如果是死亡状态，调用 interrupt 则会被忽略；
- isInterrupted，判断当前线程是否被中断，仅是对 interrupt 标识的一个判断，并不会影响标识发生任何改变；
- interrupted，是一个静态方法，虽然也用于判断当前线程是否被中断，和 isInterrupted 区别在于调用 interrupted 方法会直接擦除掉线程的 interrupt 标识。如果当前线程被打断，第一次调用 interrupted 会返回 true，并且立即擦除 interrupt 标识，之后再次调用 interrupted 则返回 false，除非线程再次被打断。

　　在源码里，isInterrupted 方法和 interrupted 方法都调用同一个本地方法：
  
```java
// ClearInterrupted 主要用来控制是否擦除线程 interrupt 标识
private native boolean isInterrupted(boolean ClearInterrupted);

// isInterrupted 方法中，参数为 false，即不擦除，这样当线程被打断一次后，多次调用 isInterrupted 都会返回 true；
public boolean isInterrupted() {
    return isInterrupted(false);
}

// 在 interrupted 方法中，参数为 true，即会擦除，所以只有第一次调用 interrupted 会返回 true
public boolean interrupted() {
    return currentThread.isInterrupted(true);
}
```

### 线程 join
　　join 方法与 sleep 一样它也是一个可中断的方法，如果有其他线程执行了对当前线程的 interrupt 操作，它也会捕获到中断信号，并且擦除线程的 interrupt 标识。<br />
　　join 某个线程 A，会使当前线程 B 进入等待，直到线程 A 结束生命周期，或者到达给定时间，在此期间线程 B 是 BLOCKED 状态的。比如主线程生成并启动了子线程，而子线程里要进行大量的耗时的运算(这里可以借鉴下线程的作用)，当主线程处理完其他的事务后，需要用到子线程的处理结果，这个时候就要用到join() 方法，让主线程等待，知道子线程完成。

### 线程关闭
　　不推荐使用 stop() 方法，已被遗弃，因该方法在关闭线程时可能不会释放掉 monitor 锁。
    
-  正常关闭；
    1. 线程结束，生命周期正常结束；
    2. 捕获中断信号关闭线程。比如 sleep、join 都为可中断方法，通过检查线程的 interrupt 标识来决定是否退出；
    3. 使用 volatile 开关控制。由于线程的 interrupt 标识可能被擦除，或者逻辑单元中不会调用任何可中断的方法，所以使用 volatile 修饰的开关 flag 关闭线程也是常用做法。
- 异常退出。线程的执行单元中，不允许抛出 check 异常，如果需要捕获，可封装成 unchecked 异常（RuntimeException）抛出进而结束线程的生命周期；
- 进程假死。假死是进程存在，但日志没有输出，程序不进行任何作业。绝大部分原因是某个线程阻塞，或线程出现死锁等情况。

### 总结：

- 线程 sleep 是一个静态方法，导致当前线程暂停指定的时间，重要一点是不会放弃 monitor 锁的所有权；
- 在闲时，设置线程优先级不会有作用，不要让业务严重依赖线程优先级；
- 使用 volatile 修饰的开关 flag 关闭线程是常用做法。
