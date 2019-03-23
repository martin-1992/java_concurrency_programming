
## 类的加载过程
　　ClassLoader 的主要是负责加载各种 class 文件到 JVM 中，ClassLoader 是一个抽象的 class，给定一个 class 的二进制文件名，ClassLoader 会尝试加载并且在 JVM 中生成构成这个类的各个数据结构，使其分布在 JVM 对应的内存区域中。<br />
　　类的加载过程围绕着二进制文件的加载、二进制数据的连接以及类的初始化进行的。
  
### 加载过程简介
　　类的加载过程分为三个阶段，分别是加载阶段、连接阶段和初始化阶段。
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_9/chapter_9_p1.png)

- 加载阶段，主要负责查找并且加载类的二进制数据文件，即 class 文件；
- 连接阶段，细分为验证、准备、解析三个阶段：
    1. 验证，主要是确保类文件的正确性，比如 class 的版本，class 文件的魔术因子是否正确；
    2. 准备，为类的静态变量分配内存，并且为其初始化默认值；
    3. 解析，把类中的符号引用转换为直接引用。
- 初始化阶段，为类的静态变量赋予正确的初始值（代码编写阶段给定的值）。

　　JVM 对类的初始化是一个延迟的机制，即使用 lazy 的方式，当一个类在首次使用时才会被初始化，在同一个运行时包下，一个 Class 只会被初始化一次。
 
 
### 类的主动使用和被动使用
　　JVM 虚拟机规定，每个类或接口被 Java 程序首次主动使用时才会对其进行初始化，同时规范以下 6 种主动使用类的场景：
  
- 通过 new 关键字的类初始化，这是最常用的初始化一个类的方式，会导致类的加载并最终初始化；
- 访问类的静态变量，包括读取和更新会导致类的初始化，如下代码中 x 是一个简单的静态变量，其他类即使不对 Simple 进行 new 创建，直接访问变量 x 也会导致类的初始化；

```java
public class Simple {
    static {
        System.out.println("initialized");
    }
    public static int x = 10;
}
```

- 访问类的静态方法，会导致类的初始化，如下代码，在其他类中直接调用 test 静态方法也会导致类的初始化；
```java
public class Simple {
    static {
        System.out.println("initialized");
    }
    
    // 静态方法
    public static void test();
}
```

- 对某个类进行反射操作，会导致类的初始化，如下运行下面代码，同样会看到静态代码块中的输出语句执行；
```java
public static void main(String[] args) throws ClassNotFoundException {
    Class.forName("com.martin.chapter09.Simple");
}
```

- 初始化子类会导致父类的初始化，如下代码，在 ActiveLoadTest 中，调用了 Child 的静态变量，根据前面讲的访问静态变量会导致类初始化，这里 Child 类被初始化，Child 类又是 Parent 类的子类，子类的初始化会进一步导致父类的初始化。注意，通过子类使用父类的静态变量只会导致父类的初始化，子类不会被初始化；

```java
public class Parent {
    static {
        System.out.println("The parent is initialized");
    }
    
    public static int y = 100;
}

public class Child extends Parent {
    static {
        System.out.println("The child will be initialized");
    }

    public static int x = 10;
}

public class ActiveLoadTest {
    public static void main(String[] args) {
        System.out.println(Child.x);
    }
}
```

- 启动类，也就是执行 main 函数所在的类会导致该类的初始化，比如使用 Java 命令运行上文中的 ActiveLoadTest 类。

　　**除了上述 6 种情况，其余的都称为被动使用，不会导致类的加载和初始化。**关于类的主动引用和被动引用，下面几个为容易引起混淆的例子：
  
- 构造某个类的数组时并不会导致该类的初始化，如下代码中 new 方法新建了一个 Simple 类型的数组，但它并不能导致 Simple 类的初始化，因此它是被动使用，而 new 关键字只是在堆内存中开辟了一段连续的地址空间 4byte x 10；

```java
public static void main(String[] args) {
    Simple[] simples = new Simple[10];
    System.out.println(simples.length);
}
```

- 引用类的静态常量不会导致类的初始化，如下代码中，MAX 是一个被 final 修饰的静态变量，也就是静态常量，在其他类中直接访问 MAX 不会导致 GlobalConstants 的初始化，虽然它也是被 static 修饰的，但如果在其他类中访问 RANDOM 则会导致类的初始化，因为 RANDOM 是需要进行随机函数计算的，在类的加载、连接阶段是无法对其进行计算的，需要进行初始化后才能对其赋予准确的值。

```java
public class GlobalConstants {
    static {
        System.out.println("The GlobalConstants will be initialized.");
    }
    
    // 在其他类中使用 MAX 不会导致 GlobalConstants 的初始化，静态代码块不会输出
    public final static int MAX = 100;
    
    // 虽然 RANDOM 是静态常量，但由于计算复杂，只有初始化之后才能得到结果，因此在其他类中使用 RANDOM 会导致 GlobalConstants 的初始化
    public final static int RANDOM = new Random().nextInt();
}
```

### 类的加载过程详解
　　简单说，类的加载就是将 class 文件中的二进制数据读取到内存中，将该字节流所代表的静态存储结构转换为方法区中运行时的数据结构，并且在堆内存中生成一个该类的 java.lang.Class 对象，作为访问方法区数据结构的入口，如下图：
  
![Aaron Swartz](https://raw.githubusercontent.com/martin-1992/java_concurrency_programming/master/chapter_9/chapter_9_p2.png)

　　类加载的最终产物就是堆内存中的 class 对象，对同一个 CLassLoader 来讲，不管某个类被加载多少次，对应到堆内存中的 class 对象始终是同一个。虚拟机规范中指出类的加载是通过一个全限定名（包名 + 类名）来获取二进制数据流，比如常见的 class 二进制文件的形式，还有以下几种：
  
- 运行时动态生成，比如通过开源的 ASM 包可生成一些 class，或通过动态代理 java.lang.Proxy 也可生成代理类的二进制字节流；
- 通过网络获取，比如很早之前的 Applet 小程序，以及 RMI 动态发布等；
- 通过读取 zip 文件获得类的二进制字节流，比如 jar、war（其实，jar 和 war 使用的是和 zip 同样的压缩算法）；
- 运行时生成 class 文件，并且动态加载，比如使用 Thrift、AVRO 等都是可以在运行时将某个 Schema 文件生成对应的若干个 class 文件，然后再进行加载。

　　注意这里所说的加载是类加载过程中的第一个阶段，并不代表整个类已经加载完成，在某个类完成加载阶段后，虚拟机会将这些二进制字节流按照虚拟机所需的格式存储在方法区中，然后形成特定的数据结构，随之又在堆内存中实例化一个 java.lang.Class 类对象，在类加载的整个生命周期中，加载过程还没有结束，连接阶段是可以交叉工作的，比如连接阶段验证字节流信息的合法性，但总体来讲加载阶段肯定是出现在连接阶段之前的。

#### 类的连接阶段
　　类的连接阶段可细分为三个小过程，分别为验证、准备和解析。<br />
　　验证在连接阶段中的主要目的是确保 class 文件的字节流所包含的内容符合当前 JVM 的规范要求，并且不会出现危害 JVM 自身安全的代码，当字节流的信息不符合要求时，则会抛出 VerifyError 这样的异常或者是其子异常，验证包含几方面信息：
  
- 验证文件格式；
    1. 验证文件的魔术因子。在很多二进制文件中，文件头部存在着魔术因子，该因子决定了这个文件到底是什么类型，class 文件的魔术因子是 0xCAFEBABE；
    2. 验证主次版本号。Java 的版本是在不断升级的，JVM 规范同样在不断升级。比如用高版本的 JDK 编译的 class 就不能够被低版本的 JVM 所兼容，在验证的过程中，还需要查看当前的 class 文件版本是否符合当前 JDK 所处理的范围；
    3. 验证构成 class 文件的字节流是否存在残缺或其他附加信息，主要看 class 的 MD5 指纹（每一个类在编译阶段经过 MD5 摘要算法计算后，都会将结果一并附加给 class 字节流作为字节流的一部分）；
    4. 常量池中的常量是否存在不被自持的变量的类型，比如 int64；
    5. 指向常量中的引用是否指到了不存在的常量或该常量的类型不被支持；
    6. 其它信息。
    
- 元数据的验证。元数据的验证其实是对 class 的字节流进行语义分析的过程，整个语义分析就是为了确保 class 字节流符合 JVM 规范的要求；
    1. 检查这个类是否存在父类，是否继承了某个接口，这些父类和接口是否合法或真实存在；
    2. 检查该类是否继承了被 final 修饰的类，被 final 修饰的类是不允许被继承并且其方法也不允许被 override；
    3. 检查该类是否为抽象类，如不是，那是否实现了父类的抽象方法或接口中的所有方法；
    4. 检查方法重载的合法性，比如相同的方法名称、相同的参数但返回类型不相同，这是不允许的；
    5. 其它语义验证。
    
- 字节码验证。主要验证程序的控制流程，比如循环、分支等；
    1. 保证当前线程在程序计数器中的指令不会跳转到不合法的字节码指令中去；
    2. 保证类型的转换是合法的，比如用 A 声明的引用，不能用 B 进行强制类型转换；
    3. 保证任意时刻，虚拟机栈中的操作栈类型与指令代码都能正确被执行，比如在压栈的时候传入的是一个 A 类型的引用，在使用的时候却将 B 类型载入了本地变量表；
    4. 其它验证。
    
- 符号引用验证。主要作用是验证符号引用转换为直接引用时的合法性，保证解析动作的顺利执行。
    1. 通过符号引用描述的字符串全限定名称是否能找到相关的类；
    2. 符号引用中的类、字段、方法，是否对当前类可见，比如不能访问引用类的私有方法；
    3. 其它。

　　准备阶段，分配内存并初始化值。当一个 class 的字节流通过了所有的验证过程后，就开始为该对象的类变量，也就是静态变量，分配内存并设置初始值，类变量的内存会被分配到方法区中，不同于实例变量会被分配到堆内存中。<br />
　　解析阶段。解析是在常量池中寻找类、接口、字段和方法的符号引用，并且将这些符号引用替换成直接引用的过程。如下面的代码，ClassResolve 用到了 Simple 类，在编写程序时，可直接使用 simple 这个引用去访问 Simple 类中可见的属性及方法。但在 class 字节码中，它会被编译成相应的助记符，这些助记符称为符号引用，在类的解析过程中，助记符还需要得到进一步的解析，才能正确地找到所对应的堆内存中的 Simple 数据结构。
  
```java
public class ClassResolve {
    static Simple simple = new Simple();
    
    public static void main(String[] args) {
        System.out.println(simple);
    }
}
```

- 类接口的解析；
    1. 假设前文代码中的 Simple，不是一个数组类型，则在加载的过程中，需要先完成对 Simple 类的加载，而如果为数组类型，需要在虚拟机中生成一个能够代表该类型的数组对象，并且在堆内存中开辟一片连续的地址空间；
    2. 在类接口的解析完成后，还需要进行符号引用的验证。
    
- 字段的解析。解析所访问类或接口中的字段，解析是自下而上的，先从子类在往上到父类。在解析类或变量的时候，如果该字段不存在，或出现错误，就会抛出异常，不再进行下面的解析；
    1. 如果 Simple 类本身就包含某个字段，则直接返回这个字段的引用，当然也要对该字段所属的类提前进行类加载；
    2. 如果 Simple 类中不存在该字段，则根据继承关系自下而上，查找父类或接口的字段，找到即返回，同样需要提前对找到的字段进行类的加载过程；
    3. 如果 Simple 类中没有字段，一直找到最上层的 java.lang.Object 还是没有，则抛出 NoSuchFieldError 异常。
    
- 类方法的解析。类方法和接口方法有所不同，类方法可以直接使用该类进行调用，而接口方法必须要有相应的实现类继承才能够进行调用；
    1. 若在类方法表中发现 class_index 中索引的 Simple 是一个接口而不是一个类，则直接返回错误；
    2. 在 Simple 类中查找是否有方法描述和目标方法完全一致的方法，如果有，则直接返回这个方法的引用，否则直接继续向上查找；
    3. 如果父类中仍没有找到，则意味着查找失败，抛出 NoSuchMethodError 异常；
    4. 如果在当前类或父类中找到了和目标方法一致的方法，但它是一个抽象类，则会抛出 AbstractMethodError 这个异常。
    
　　类的初始化阶段，最主要就是执行 &lt;clinit&gt; 方法的过程（clinit 是 class initialize 的缩写），在 &lt;clinit&gt;() 方法中所有的类变量都会被赋予正确的值，即程序编写的时候指定的值。<br />
　　&lt;clinit&gt;() 方法是在编译阶段生成的，它已经包含在 class 文件中了，&lt;clinit&gt;() 中包含了所有类变量的赋值动作和静态语句块的执行代码，编译器收集的顺序是由执行语句在源文件中的出现顺序所决定的。注意，静态语句块只能对后面的静态变量进行赋值，但不能对其进行访问。<br />
　　另外 &lt;clinit&gt;() 方法与类的构造函数有所不同，它不需要显式地调用父类的构造器，虚拟机会保证父类的 &lt;clinit&gt;() 方法最先执行，因此父类的静态变量总是能够得到优先赋值。
  
```java
public class ClassInit {
    // 父类中有静态变量 value
    static class Parent {
        static int value = 10;
        // 静态语句的代码块，父类的 <clinit>() 方法优先执行
        static {
            value = 20;
        }
    }
    
    // 子类使用父类的静态变量为自己的静态变量赋值
    static class Child extends Parent {
        static int i = value;
    }
    
    public static void main(String[] args) {
        System.out.println(Child.i);
    }
}
```

　　上面程序的输出为 20，不是 10，因为父类的 &lt;clinit&gt;() 方法优先执行。注意，当某个类中既没有静态代码块，也么有静态变量，就不会生成 &lt;clinit&gt;() 方法。由于接口不能定义静态代码块，因此只有当接口中有变量的初始化操作时才会生成 &lt;clinit&gt;() 方法。<br />
　　&lt;clinit&gt;() 方法只能被虚拟机执行，在主动触发了类的初始化后会调用这个方法。而在多线程中，只能由一个线程执行到静态代码块中的内容，并且静态代码块仅仅只会被执行一次，JVM 保证了 &lt;clinit&gt;() 方法在多线程下的同步语义，因此在单例设计模式下，采用 Holder 的方式是一种最佳的设计方法。
