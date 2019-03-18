
## 线程间通信
　　线程间通信又称为进程内通信，多个线程实现互斥访问共享资源时会互相发送信号或等待信号。
    
### wait 和 notify 方法详解
　　wait 和 notify 方法并不是 Thread 特有的方法，而是 Object 中的方法，在 JDK 中的每个类都拥有这两个方法。<br />

#### wait 方法
    
- wait 方法的这三个重载方法都将调用 wait(long timeout) 这个方法，wait() 方法即 wait(0)，0 表示永不超时；
- Object 的 wait(long timeout) 方法会导致当前线程进入阻塞，直到有其他线程调用了 Object 的 notify 和 notifyAll 方法将其唤醒，或阻塞时间达到了 timeout 时间而自动唤醒；
- wait 方法必须拥有该对象的 monitor，也就是 wait 方法必须在同步方法中使用；
- 当前线程执行了该对象的 wait 方法后，会释放对 monitor 的所有权，让其他线程抢 monitor 所有权。

#### notify 方法
  
- 唤醒单个正在执行该对象 wait 方法的线程；
- 如果有某个线程由于执行该对象的 wait 方法而进入阻塞则会被唤醒，没有则忽略；
- 被唤醒的线程需要重新获取对该对象所关联的 monitor 的 lock 才能继续执行。

#### 关于 wait 和 notify 的注意事项

- wait 方法是可中断方法，当前线程调用 wait 方法进入阻塞状态，其他线程可以用 interrupt 方法进行打断。可中断方法被打断后会收到 InterruptException，同时 interrupt 标识也会被擦除；
- 线程执行了某个对象的 wait 方法后，会加入与之对应的 wait set 中，每一个对象的 monitor 都有与之关联的 wait set；
- 当线程进入 wait set 后，notify 方法可唤醒，即从 wait set 中弹出，同时中断 wait 中的线程也会将其唤醒；
- 必须在同步方法中使用 wait 和 notify 方法，因为执行 wait 和 notify 的前提条件是持有同步方法的 monitor 的所有权，并且哪个 monitor 对象进行同步，就只能用那个对象的 wait 和 notify，否则会抛出 IllegalMonitorStateException。

#### wait  和 sleep 的区别
　　表面上看，wait 和 sleep 方法都可以使当前线程进入阻塞状态，但存在本质区别：

- wait 和 sleep 方法都可以使当前线程进入阻塞状态，且都是可中断方法，被中断后会收到中断异常；
- wait 是 Object 的方法，而 sleep 是 Thread 特有的方法；
- wait 方法的执行必须在同步方法中，而 sleep 不需要；
- 线程在同步方法中执行 sleep 方法时，不会释放 monitor 锁，而 wait 会释放；
- sleep 方法短暂休眠后主动退出阻塞，因为不释放 monitor 锁，而 wait 方法（没有指定 wait 时间）则需要被其他线程中断后才能退出阻塞。

### 多线程间通信
　　多线程间通信需要用到 Object 的 notifyAll 方法，该方法与 notify 类似，都可以唤醒由于调用 wait 方法而阻塞的线程，区别是 notifyAll 是唤醒全部的阻塞线程，同样被唤醒的线程需争抢 monitor 锁。
　　
#### 线程休息室 wait set
　　线程调用某个对象的 wait 方法后，都会加入与该对象 monitor 关联的 wait set 中，并且释放 monitor 的所有权。<br />
　　如下图，若干线程调用 wait 方法后加入与 monitor 关联的 wait set 中，当另外一个线程调用该 monitor 的 notify 方法后，其中一个线程会从 wait set 中弹出。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_5/chapter_5_p1.png)

　　而 notifyAll 则将 wait set 中的所有线程弹出：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_5/chapter_5_p2.png)

### synchronized 关键字的缺陷
　　synchronized 关键字提供一种排他式的数据同步机制，某个线程在获取 monitor 锁的时候可能被阻塞，而这种阻塞有两个缺陷：
  
- 因争抢 monitor 锁造成线程阻塞，无法控制阻塞时长，会一直阻塞，直到获取锁。假设有两个线程 t1 和 t2，t1 线程进入 sleep 阻塞状态为 1 小时，t2 线程要获得 monitor 锁，只能等 t1 线程退出阻塞状态；
- 因争抢 monitor 锁进入阻塞状态，不可被中断。当 t2 线程在争抢 t1 线程的锁进入阻塞状态时，t2 线程无法被中断。
