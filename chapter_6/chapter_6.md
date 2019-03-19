
## ThreadGroup 详解
　　创建线程时，如果没有显式指定 ThreadGroup，新的线程会被加入与父线程相同的 ThreadGroup 中，即新的线程会加入到 main 线程所在的 group 中，而 main 线程的 group 名为 main。<br />
　　如同线程存在父子关系一样，ThreadGroup 同样存在父子关系，如下图为父子 thread、父子 threadGroup 以及 thread 和 group 之间的层次关系：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_6/chapter_6_p1.png)

　　无论如何，线程都会加入到某个 ThreadGroup 之中。
  
### 创建 ThreadGroup
　　第一个构造函数为 ThreadGroup 赋予名字，但是该 ThreadGroup 的父 ThreadGroup 是创建它的线程所在的 ThreadGroup。第二个 ThreadGroup 的构造函数赋予 group 名字的同时又显式指定父 group。

```java
public ThreadGroup(String name) 
public ThreadGroup(ThreadGroup parent, String name)
```

### 复制 Thread 数组和 ThreadGroup 数组
　　下面两个方法，将 ThreadGroup 中的 active 线程全部复制到 Thread 数组中，其中 recurse 参数为 true，则该方法会将所有子 group 中的 active 线程都递归到 Thread 数组中，enumerate(Thread[] list) 实际上等价于 enumerate(Thread[] true)，这两方法都调用了 ThreadGroup 的私有方法 enumerate：
    
```java
public int enumerate(Thread[] list)
public int enumerate(Thread[] list, boolean, recurse)


private int enumerate(Thread list[], int n, boolean recurse) {
    int ngroupsSnapshot = 0;
    ThreadGroup[] groupsSnapshot = null;
    synchronized (this) {
        // ThreadGroup 销毁，则返回 0
        if (destroyed) {
            return 0;
        }
        int nt = nthreads;
        // 线程数超过列表的容量，则以列表容量为准
        if (nt > list.length - n) {
            nt = list.length - n;
        }
        // 遍历线程数组，如果为活跃线程，则加入到列表中
        for (int i = 0; i < nt; i++) {
            if (threads[i].isAlive()) {
                list[n++] = threads[i];
            }
        }
        // 
        if (recurse) {
            ngroupsSnapshot = ngroups;
            if (groups != null) {
                groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
            } else {
                groupsSnapshot = null;
            }
        }
    }
    if (recurse) {
        for (int i = 0 ; i < ngroupsSnapshot ; i++) {
            n = groupsSnapshot[i].enumerate(list, n, true);
        }
    }
    return n;
}
```
    
　　注意，enumerate 方法获取的线程仅仅是个预估值，并不能百分百保证当前 group 的活跃线程，比如在调用复制后，某个线程结束生命周期或加入了新线程，都会导致数据不准。<br />
　　和复制 Thread 数组类似，如下两个方法用于复制当前 ThreadGroup 的子 Group，recurse 决定是否以递归的方式复制：
  
```java
public int enumerate(ThreadGroup[] list)
public int enumerate(ThreadGroup[] list, boolean recurse)
```

### ThreadGroup 操作

- activeCount()，获取 group 中活跃的线程，只是个估计值，前面有分析，该方法会递归获取其他子 group 中的活跃线程；
- activeGroupCount() ，获取 group 中活跃的子 group，也是估计值；
- getMaxPrority()，获取 group 的优先级，默认为 10，所有线程的优先级都不能大于 group 的优先级；
- getName()，获取 group 的名字；
- getParent()，获取 group 的父 group，如果父 group 不在，则返回 null，比如 system group 的父 group 为 null；
- list()，将 group 中所有活跃线程信息全部输出到控制台，即 System.out；
- parentOf(ThreadGroup g)，判断当前 group 是不是给定 group 的父 group，另外如果给定的 group 是自己本身，也会返回 true；
- setMaxPriority(int pri)，指定 group 的最大优先级，但不能超过父 group 的最大优先级，执行该方法不仅会改变当前 group 的最大优先级，还会改变所有子 group 的最大优先级。

#### ThreadGroup 的 interrupt
　　interrupt 一个 thread group 会导致该 group 中所有的 active 线程都被 interrupt，即该 group 中每一个线程的 interrupt 标识都被设置。如下为 ThreadGroup interrupt 方法的源码：
  
```java
public final void interrupt() {
    int ngroupsSnapshot;
    ThreadGroup[] groupsSnapshot;
    synchronized (this) {
        // 检查当前线程是否允许修改线程组
        checkAccess();
        // 遍历对线程组中的每个线程进行 interrupt
        for (int i = 0 ; i < nthreads ; i++) {
            threads[i].interrupt();
        }
        ngroupsSnapshot = ngroups;
        if (groups != null) {
            groupsSnapshot = Arrays.copyOf(groups, ngroupsSnapshot);
        } else {
            groupsSnapshot = null;
        }
    }
    // 递归获取子 group，执行 interrupt 方法
    for (int i = 0 ; i < ngroupsSnapshot ; i++) {
        groupsSnapshot[i].interrupt();
    }
}
```

#### ThreadGroup 的 destroy
　　destroy 用于销毁 ThreadGroup，该方法针对一个没有任何 active 线程的 group 进行一次 destroy 标记，调用该方法的直接结果是在父 group 中将自己移除，如果有 active 线程，则会抛出异常。
  
#### 守护 ThreadGroup
　　线程可以设置为守护线程，ThreadGroup 也可以设置为守护 ThreadGroup，将 ThreadGroup 设置为 daemon，不会影响线程的 daemon 属性。设置为守护 ThreadGroup 后，当 group 中没有任何 active 线程时，该 group 会自动 destroy。
