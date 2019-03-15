
## 线程安全与数据同步
　　共享资源指的是多个线程同时对同一份资源进行访问（读写操作），被多个线程访问的资源称为共享资源。如何保证多个线程访问到的数据是一致的，则被称为数据同步或资源同步。
  
### synchronized 关键字
　　synchronized 提供一种排他机制，即同一时间只有一个线程执行某些操作。
　　
- synchronized 关键字提供一种锁的机制，能够确保共享变量的互斥访问，从而防止数据不一致性的问题；
- synchronized 关键字包含 monitor enter 和 monitor exit 两个 JVM 指令，能够保证在任何时候任何线程执行到 monitor enter 成功之前都必须从主内存中获取数据，而不是从缓存中，在 monitor exit 运行成功后，共享变量被更新后的值必须刷入主内存；
- synchronized 指令严格遵守 java happens-before 规则，一个 monitor exit 指令之前必须要有一个 monitor enter。

　　synchronized 可用于对代码块或方法进行修饰，但不能用于对 class 或变量进行修饰，而 volatile 则对类变量进行修饰。<br />

#### 线程堆栈分析
　　synchronized 关键字提供了一种互斥机制，也就是说在同一时刻，只有一个线程访问同步资源，即该线程获取了与 mutex 关联的 monitor 锁。如下图，Thread-2 持有 monitor&lt;0x000000078b936ce0&gt;的锁，并且处于休眠状态中，其他线程需等待 Thread-2 释放锁资源。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_4/chapter_4_p1.png)

#### JVM 指令分析

- Monitorenter，每个对象都与一个 monitor 相关联，一个 monitor 的 lock 锁只能被一个线程在同一时间获得，一个线程尝试获得与对象关联的 monitor 所有权时：
    1. monitor 计数为 0，表示该 monitor 的 lock 还没有被获得，某个线程获得后就是这个 monitor 的持有者，并且该计数器加一；
    2. 如果一个已经拥有该 monitor 所有权的线程重入，会导致 monitor 计数器再次累加；
    3. 如果 monitor 已经被其他线程所拥有，当其他线程尝试获取该 monitor 时，会进入阻塞状态，直到该 monitor 的计数器变为 0，才能再次尝试获取对 monitor 的所有权。
    
- Monitorexit，释放对 monitor 的所有权，就是将 monitor 的计数器减一，如果计数器的结果为 0 就解锁，释放的前提是曾经获得所有权。 

#### 使用 synchronized 需注意的问题

- 与 monitor 关联的对象不能为空，如下 mutex 为空是错误的；

```java
private final Object mutex = null;
public void syncMethod() {
    synchronized (mutex) {
        // 
    }
}
```

- synchronized 作用域太大。由于 synchronized 关键字存在排他性，所有线程必须串行经过 synchronized 所保护的共享区域，如果 synchronized 所保护的区域越大，则代表效率越低，应尽可能只用于共享资源（数据）的读写作用域；
- 不同的 monitor 企图锁相同的方法；
- 多个锁的交叉导致死锁。

### 程序死锁
　　程序死锁，可类比如下图所示的交通堵塞。
    
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_4/chapter_4_p2.png)

　　导致程序死锁的情况：
  
- 交叉锁可导致程序出现死锁。线程 A 持有 R1 的锁等待获取 R2 的锁，线程 B 持有 R2 的锁等待获取 R1 的锁；
- 内存不足。当并发请求系统可用内存时，如果此时系统内存不足，则可能出现死锁的情况。举例，两个线程 T1 和 T2，执行某个任务，其中 T1 获取了 10MB 内存，T2 获取了 20MB 内存。如果每个线程的执行单元需要 30MB 内存，但剩余可用内存为 20MB，则两个线程有可能等待彼此能够释放内存资源；
- 一问一答式的数据交换。服务端开启某个窗口，等待客户端访问，客户端发送请求立即等待接收。由于某种原因服务端错过了客户端的请求，仍然在等待一问一答式的数据交换，此时服务端和客户端都在等待双方发送数据；
- 数据库锁。无论是数据库表级别的锁，还是行级别的锁，比如某个线程执行 for update 语句退出了事务，其他线程访问该数据库时将陷入死锁；
- 文件锁。同理，某线程获得文件锁意外退出，其他读取该文件的线程也将进入死锁，直到系统释放文件句柄资源；
- 死循环引起的死锁。进入死循环，虽然查看线程堆栈信息不会发现任何死锁迹象，但程序不工作，CPU 占有率又居高不下，这种死锁一般称为系统假死，比如使用非线程安全类 HashMap。
