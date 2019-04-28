
## Single Thread Execution 设计模式
　　Single Thread Execution 模式采用排他式的操作，保证在同一时刻只能有一个线程去访问共享资源。
  
　　使用 Single Thread Execution 场景：
  
- 多线程访问资源的时候，被 synchronized 同步的方法总是排他性的；
- 多个线程对某个类的状态发生改变的时候，比如赋值操作。

　　将某个类设计成线程安全的类，用 Single Thread Execution 控制是其中的方法之一，但是子类如果继承了线程安全的类并且打破了 Single Thread Execution 的方式，会破坏方法的安全性，称为继承异常。<br />
　　在 Single Thread Execution 中，synchronized 关键字起到了决定性作用，但是 synchronized 的排他性是以牺牲性能为代价，因此在保证线程安全的前提下应尽量缩小 synchronized 的作用域。
