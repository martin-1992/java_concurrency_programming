
## JVM 类加载器
　　类的加载器就是负责类的加载职责，对于任意一个 class，都需要由加载它的类加载器和这个类本身确立其在 JVM 中的唯一性，这也是运行时包。
  
### JVM 内置三大类加载器
　　不同的类加载器负责将不同的类加载到 JVM 内存中，并且它们之间遵循父委托机制。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_10/chapter_10_p1.png)

- BootStrap ClassLoader（根加载器），最顶层的加载器，没有任何父加载器，主要负责虚拟机核心类库的加载，java.lang 包是由根加载器所加载，保证能正确载入（父委托机制）；
- Ext ClassLoader（扩展类加载器），父加载器为 BootStrap ClassLoader，用于加载 JAVA_HOME 下的 jre/lb/ext 子目录里面的类库，是 java.lang.URLClassLoader 的子类，将自己的类打包成 jar 包，放到扩展类加载器所在的路径下，即可被加载；
- Application ClassLoader（应用加载器，又称系统类加载器），常见的类加载器，父加载器为扩展类加载器，负责加载 classpath 下的类库资源，为自定义加载的默认父加载器。

### 自定义类加载器
　　所有的自定义类加载器都是 ClassLoader 的直接子类或间接子类，java.lang.ClassLoader 是一个抽象类，需要实现 findClass 方法。<br />
　　在定义 ClassLoader 之前，要强调 defineClass 方法，该方法的完整方法描述是 defineClass(String name, byte[] b, int off, int len)。
- String name，是要定义类的名字，一般与 findClass 方法中的类名保持一致；
- byte[] b，class 文件的二进制字节数组；
-  int off，字节数组的偏移量；
- int len，从偏移量开始读取多长的  byte 数据。偏移量和读取长度是为了能选取指定 class，因为 一个字节数组可能存储多个 class 的字节信息。

### 双亲委托机制介绍
　　又叫父委托机制。当一个类加载器被调用了 loadClass 之后，并不是直接加载，而是先交给当前类加载器的父加载器尝试加载，直到最顶层的父加载器，然后再依次向下加载。双亲委托机制是一种包含关系，而非继承关系。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_10/chapter_10_p2.png)

```java
public Class<?> loadClass(String name) throws ClassNotFoundException {
    return loadClass(name, false);
}

protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 从当前类加载器的已加载类缓存中根据类的全路径名查询是否存在该类，如果存在则直接返回
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                // 如果当前类存在父类加载器，则调用父类加载器的 loadClass(name, false) 方法对其进行加载，
                // resolve=false，所以不会进行连接阶段的继续执行，表示通过类加载器不会导致类的初始化
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    // 如果当前类加载器，不存在父类加载器，则直接调用根类加载器对该类进行加载
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // 如果当前类的所有父类加载器都没有成功加载 class，则尝试调用当前类加载器的 findClass 方法对其
                // 进行加载，该方法是我们自定义加载器需要重写的方法
                long t1 = System.nanoTime();
                c = findClass(name);

                // 当类被加载成功，则做性能统计
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

### 类加载器命名空间
　　每一个类加载器都有各自的命名空间，命名空间是由该加载器及其所有父加载器所构成的，因此在每个类加载器中同一个 class 都是唯一的，即使用 classLoader.loadClass("xx")，都是同一个对象。如下图为类被加载后的内存情况：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_10/chapter_10_p3.png)

　　注意，使用不同的类加载器，或同一个类加载器的不同实例，去加载同一个 class，则会在堆内存和方法区产生多个 class 的对象。如下图，同一个 class 被不同类加载器加载后的内存情况：

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_10/chapter_10_p4.png)

　　总结，同一个 class 实例在同一个类加载器命名空间之下是唯一的。
  
### 运行时包
　　在 JVM 运行时 class 会有一个运行时包，是由类加载器的命名空间和类的全限定名称共同组成的，举例：
  
```java
BoostrapClassLoader.ExtClassLoader.AppClassLoader.MyClassLoader.com.martin.concurrent.chapter10.Test
```

　　这样做的目的是为了安全和封装，防止不同包下同样名称的 class 引起冲突。
  
### 初始类加载器
　　由于运行时包的存在，JVM 规定了不同的运行时包下的类彼此之间是不可以进行访问的，但在开发的程序中可以访问 java.lang 包下的类，

- 每个类在通过 ClassLoader 加载后，在虚拟机中有对应的 Class 实例，比如某个类 C 被类加载器 CL 加载，那么 CL 为 C 的初始类加载器；
- 在 JVM 中，每个类加载器都有一个列表，记录了将该类加载器作为初始类加载器的所有 class。在加载一个类时，JVM 会先判断该类是否已经在列表中，即是否被加载过；
- 根据 JVM 规定，在类的加载过程中，所有参与的类加载器，即使没有亲自加载过该类，也都会被标识为该类的初始类加载器，比如 java.lang.String 首先经过了 BrokerDelegateClassLoader 类加载器，又经过了系统加载器、扩展类加载器、根类加载器，这些类加载器都是 java.lang.String 的初始类加载器，JVM 会在每一个类加载器维护的列表中添加该 class 类型；

![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_10/chapter_10_p5.png)

- 虽然 SimpleClass 和 java.lang.String 由不同的类加载器加载，但是在 BrokerDelegateClassLoader 的 class 列表中维护了 SimpleClass.class 和 String.class，因此在 SimpleClass 中可以正常访问 rt.jar 中的 class 的。

### 类的卸载
　　-verbose:class 可查看 JVM 运行时加载多少 class，一个 Class 需满足下列条件才会被 GC 回收：

- 该类的所有实例都被 GC；
- 加载该类的 ClassLoader 实例被回收；
- 该类的 class 实例没有被引用。
