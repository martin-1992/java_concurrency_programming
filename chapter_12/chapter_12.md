
## volatile 关键字介绍
　　volatile 关键字只能修饰类变量和实例变量，对于方法参数、局部变量、实例常量以及类常量都不能修饰。

### Java 内存模型
　　Java 内存模型（Java Memory Mode，JMM）指定了 Java 虚拟机如何与计算机的主存（RAM）进行工作。Java 的内存模型决定了一个线程对共享变量的写入何时对其他线程可见，Java 内存模型定义了线程和主内存之间的抽象关系。如下：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_12/chapter_12_p1.png)
  
- 共享变量存储于主内存中，每个线程都可以访问；
- 每个线程都有私有的工作内存或者称为本地内存；
- 工作线程只存储该线程对共享变量的副本；
- 线程不能直接操作主内存，只有先操作了工作内存后才能写入主内存；
- 工作内存和 Java 内存模型一样也是一个抽象概念，它其实并不存在，涵盖了缓存、寄存器、编译器优化以及硬件等。
