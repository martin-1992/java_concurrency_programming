
## 监控任务的生命周期
　　Thread 提供可获取状态，以及判断是否 alive 的方法，但是都是针对线程本身的，而提交的任务 Runnable 在运行过程中所处的状态如何是无法直接获得的。
  
### 观察者模式与 Thread
　　当某个对象发送状态改变需要通知第三个时，适合使用观察者模式。观察者模式的主体为 Thread，负责执行任务的逻辑单元，清楚整个过程。事件的接收者则是通知接受者的一方，严格意义上的观察者模式需要 Observer 的集合，这里只需将执行任务的每一个阶段都通知给观察者即可。

### 接口定义

#### Observable 接口定义
　　该接口主要是暴露给调用者使用的，四个枚举类型分别代表了当前任务执行生命周期的各个阶段：
  
- getCycle() 方法，用于获取当前任务处理哪个执行阶段；
- start() 方法，屏蔽 Thread 类其他的 api，通过 Observable 的 start 对线程进行启动；
- interrupt() 方法，作用域 start 一样，通过 Observable 的 interrupt 对当前线程进行中断。

```java
public interface Observable {
    // 任务生命周期的枚举类型
    enum Cycle {
        STARTED, RUNNING, DONE, ERROR
    }
    
    // 获取当前任务的生命周期状态
    Cycle getCycle();
    
    // 定义启动线程的方法，主要是为了屏蔽 Thread 的其他方法
    void start();
    
    // 定义线程的打断方法，也是为了屏蔽 Thread 的其他方法
    void interrupt();
}
```

#### TaskLifecycle 接口定义
　　TaskLifecycle 接口定义了在任务执行的生命周期中会被触发的接口，其中 Empty-Lifecycle 是一个空的实现，为的是让使用者保持对 Thread 类的使用习惯。


```java
public interface TaskLifecycle<T> {
    // 任务启动时触发 onStart 方法
    void onStart(Thread thread);
    
    // 任务正在运行时触发 onRunning 方法
    void onRunning(Thread thread);
    
    // 任务运行结束时触发 onFinish 方法，其中 result 是任务执行结束后的结果
    void onFinish(Thread thread, T result);
    
    // 任务执行报错时触发 onError 方法
    void onError(Thread thread, Exception e);
    
    // 生命周期接口的空实现（Adapter）
    class EmptyLifecycle<T> implements TaskLifecycle<T> {
        @Override
        public void onStart(Thread thread) {
            // do nothing
        }
        
        @Override
        public void onRunning(Thread thread) {
        }
        
        @Override
        public void onFinish(Thread thread) {
        }
        
        @Override
        public void onError(Thread thread) {
        }
    }
}
```

#### Task 函数接口定义

```java
@FunctionalInterface
public interface Task<T> {
    // 任务执行接口，该接口允许有返回值
    T call();
}
```

　　由于需要对线程中的任务执行增加可观察的能力，并且需要获得最后的计算结果。因此 Runnable 接口在可观察的线程中将不再使用，使用 Task 接口，其作用域 Runnable 类似，用于承载任务的逻辑执行单元。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_15/chapter_15_p1.png)

#### ObservableThread 实现
　　ObservableThread 是任务监控的关键，继承自 Thread 类和 Observable 接口，并且在构造期间需要传入 Task 的具体实现：
  
```java
public class ObservableThread<T> extends Thread implements Observable {
    
    private final TaskLifecycle<T> lifecycle;
    
    private final Task<T> task;
    
    private Cycle cycle;
    
    // 指定 Task 的实现，默认情况下使用 EmptyLifecycle
    public ObservableThread(Task<T> task) {
        this(new TaskLifecycle.EmptyLifecycle<>(), task);
    }
    
    // 指定 TaskLifecycle 的同时指定 Task
    public ObservableThread(TaskLifecycle<T> lifecycle, Task<T> task) {
        super();
        // Task 不允许为 null
        if (task == null) {
            throw new IllegalArgumentException("The task is required.");
        }
        this.lifecycle = lifecycle;
        this.task = task;
    }
    
    @Override
    public final void run() {
    }
}


```
