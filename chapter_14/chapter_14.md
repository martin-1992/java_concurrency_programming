
## 7 种单例设计模式的设计
　　单例设计模式提供一种在多线程情况下保证实例唯一性的解决方案。
　　
### 饿汉式
　　饿汉式的关键在于 instance 作为类变量并且直接得到初始化。
  
- 优点，百分百同步，即在多线程的情况下不可能被实例化两次，因为 instance 作为类变量在类初始化的过程中会被收集进 &lt;clinit&gt;() 方法中；
- 缺点，由于是直接被 ClassLoader，无法再需要时在加载（懒加载），所以可能在堆内存中占用很长时间，才被使用。

```java
// final 不允许被继承
public final class Singleton {
    // 实例变量
    private byte[] data = new byte[1024];
    
    // 在定义实例对象的时候直接初始化
    private static Singleton instance = new Singleton();
    
    // 私有构造函数，不允许外部 new
    private Singleton {
    
    }
    
    public static Singleton getSingleton() {
        return instance;
    }
    
}
```

　　总结，饿汉式的单例设计模式可保证多个线程下的唯一实例，但无法进行懒加载。

### 懒汉式
　　懒汉式是在使用类实例的时候再去创建（用到才创建），可避免类在初始化时提前创建（饿汉式）。
  
- 优点，初始化时不会被实例化，只有在需要时使用 getInstance 方法中判断为空时，才实例化；
- 缺点，无法保证单例的唯一性，可能出现在多线程情况下，多个线程刚好同时看到 instance 为 null，导致创建多个实例。

```java
// final 不允许被继承
public final class Singleton {
    // 实例变量
    private byte[] data = new byte[1024];
    
    // 定义实例，但是不直接初始化
    private static Singleton instance = null;
    
    private Singleton() {
    
    }
    
    public static Singleton getInstance() {
        // 为空时，才初始化创建
        if (null == instance) {
            instance = new Singleton();
        }
        return instance;
    }
    
}
```


### 懒汉式 + 同步方法

- 优点，保证 instance 实例的唯一性；
- 缺点，性能低下，因 synchronized 关键字天生的排他性导致了 getInstance 方法只能在同一时刻被一个线程所方法。

```java
// final 不允许被继承
public final class Singleton {
    // 实例变量
    private byte[] data = new byte[1024];
    
    private static Singleton instance = null;
    
    private Singleton() {
    
    }
    
    // 向 getInstance 方法中加入同步控制，每次只能有一个线程进入
    public static synchronized Singleton getSingleton() {
        if (null == instance) {
            instance = new Singleton();
        }
    return instance;
    }
}
```




### Double-check

- 优点，保证 instance 实例的唯一性，兼顾性能，即只有在第一次判断为 null，才使用 synchronized；
- 缺点，在多线程情况下有可能会引起空指针异常，即类成员的实例化 conn 和 socket 发生在 instance 之后。

```java
// final 不允许被继承
public final class Singleton {
    // 实例变量
    private byte[] data = new byte[1024];
    
    private static Singleton instance = null;
    
    Connection conn;
    
    Socket socket;
    
    private Singleton() {
        // 初始化 conn
        this.conn;
        // 初始化 socket
        this.socket;
    }
    
    public static Singleton getInstance() {
        // 当 instance 为 null 时，进入同步代码块，同时该判断避免每次都需要进入同步代码块
        if (null == instance) {
            // 只有一个线程能够获得 Singleton.class 关联的 monitor 
            synchronized (Singleton.class) {
                // 判断如果 instance 为 null 则创建
                if (null == instance) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
    
}
```

　　如下图，在 Singleton 的构造函数中，需要分别实例化 conn 和 socket 两个资源，还有 Singleton 自身。根据 JVM 运行时指令重排序和 Happens-Before 规则，这三者间的实例化顺序无前后约束关系。<br />
　　于是，可能出现 instance 最先被实例化，而 conn 和 socket 未完成实例化，这时另外一个线程判断 instance 已被实例化，于是调用未完成实例化的 conn 和 socket，导致出现空指针。

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_14/chapter_14_p1.png)

### Volatile + Double-Check
　　防止 Double-Check 出现空指针异常，因 volatile 关键字可防止重排序，就保证了类成员变量实例化完，才实例化（Singleton）自身。满足多线程下的单例，懒加载以及获取实例的高效性。


```java
// final 不允许被继承
public final class Singleton {
    // 实例变量
    private byte[] data = new byte[1024];
    
    private volatile static Singleton instance = null;
    
    Connection conn;
    
    Socket socket;
    
    private Singleton() {
        this.conn;
        this.socket;
    }
    
    public static Singleton getInstance() {
        if (null == instance) {
            synchronized (Singleton.class) {
                if (null == instance) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
    
}
```

### Holder 方式
　　Holder 的方式是借助了类加载的特点，在 Singleton 类中并没有 instance 的静态成员，而是将其放到了静态内部类 Holder 之中，因此在 Singleton 类的初始化过程中并不会创建 Singleton 的实例，Holder 类中定义了 Singleton 的静态变量，并且直接进行了实例化。<br />
　　当 Holder 被主动引用的时候则会创建 Singleton 的实例，Singleton 实例的创建过程在 Java 程序编译时期收集至 &lt;clinit&gt;() 方法中，该方法又是同步方法，可保证内存的可见性、JVM 指令的顺序性和原子性，是目前使用较广的设计之一。
  
```java
// 不允许被继承
public final class Singleton {
    // 实例变量
    private byte[] data = new byte[1024];
    
    private Singleton() {
    
    }
    
    // 在静态内部类中持有 Singleton 的实例，并且可被直接初始化
    private static class Holder {
        private static Singleton instance = new Singleton();
    }
    
    // 调用 getInstance 方法，事实上是获得 Holder的 instance 静态属性
    public static Singleton getInstance() {
        return Holder.instance;
    }
}
```



### 枚举方式
　　很多开源代码使用枚举方式实现单例模式，枚举类型不允许被继承，保证线程安全（底层做了线程安全的保证，无需自己实现线程安全）和反序列化安全性，且只能被实例化一次，但枚举类型不能够懒加载。
  
```java
// 枚举类型本身是 final，不允许被继承
public enum Singleton {
    
    INSTANCE;
    
    // 实例变量
    private byte[] data = new byte[1024];
    
    Singleton () {
        System.out.println("INSTANCE will be initialized immediately");
    }
    
    public static void method() {
    // 调用该方法会主动使用 Singleton，INSTANCE 将会被实例化
    }
    
    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

　　可对其进行改造，增加懒加载的特性，类似于 Holder的方式。
  
```java
public class Singleton {
    // 实例变量
    private byte[] data = new byte[1024];
    
    private Singleton() {
    
    }
    
    // 使用枚举充当 holder
    private enum EnumHolder {
        INSTANCE;
        
        private Singleton instance;
        
        EnumHolder() {
            this.instance = new Singleton();
        }
        
        private Singleton getSingleton() {
            return instance;
        }
    }
    
}
```
