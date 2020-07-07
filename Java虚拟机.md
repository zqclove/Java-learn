# Java虚拟机

Java设计理念是“write once，run anywhere”。

Java编译器将 Java源文件 - A.java文件编译成 Java字节码文件 - A.class文件。.class文件由 JVM 进行解释。

[![Run Java bytecode on different platforms - the "write once, run anywhere" concept](https://javatutorial.net/wp-content/uploads/2017/10/write-once-run-anywhere-jvm.png)](https://javatutorial.net/wp-content/uploads/2017/10/write-once-run-anywhere-jvm.png)

​	字节码是 Java 和 机器语言之间的中间语言。相同的字节码文件可以被任何 JVM 实现上执行，不需要因为平台不同而调整代码。

## Java虚拟机架构

JVM 主要由三个子系统组成：

- 类加载器子系统
- 运行时数据区
- 执行引擎

[![Java Virtual Machine architecture diagram ](https://javatutorial.net/wp-content/uploads/2017/10/jvm-architecture-992x1024.png)](https://javatutorial.net/wp-content/uploads/2017/10/jvm-architecture.png)



# 类加载子系统

## 概述

类加载器是 JVM 的重要组成部分，被用于加载类和接口。为方便 Java开发人员的日常工作，无需自定义类加载器，JVM已经为开发人员写好了可用的类加载器。当然，如果为了满足某些需求，也可自定义类加载器。例如Tomcat容器，Tomcat为每个Web应用程序创建一个类加载器（这样它就可以卸载Web应用程序并释放内存）。

## 什么是类加载器

​	Java程序是在 JVM 中运行。当我们编译一个 Java 类时，它会被转换为一个独立于平台与机器的字节码文件。当我们试图使用一个类时，Java 类加载器会将这个类加载到内存中。当在一个已经运行的类中按名称引用时，类会被引入 Java 环境。在第一个类运行之后，类装入器将尝试装入类。运行第一个类通常是通过声明和使用静态main()方法来完成的。

[![hierarchy of class loaders](https://javatutorial.net/wp-content/uploads/2017/10/hierarchy-of-class-loaders.png)](https://javatutorial.net/wp-content/uploads/2017/10/hierarchy-of-class-loaders.png)



## 类加载器类型

**从 Java 虚拟机的角度来讲，只存在以下两种不同的类加载器：**

- 启动类加载器（Bootstrap ClassLoader），使用 C++ 实现，是虚拟机自身的一部分；
- 所有其它类的加载器，使用 Java 实现，独立于虚拟机，继承自抽象类 java.lang.ClassLoader。

**从 Java 开发人员的角度看，类加载器可以划分得更细致一些：**

1. **启动类加载器（Bootstrap ClassLoader）：**

   ​	此类加载器负责将存放在 <JRE_HOME>\lib 目录中的，或者被 -Xbootclasspath  参数所指定的路径中的，并且是虚拟机识别的（仅按照文件名识别，如 rt.jar，名字不符合的类库即使放在 lib  目录中也不会被加载）类库加载到虚拟机内存中。启动类加载器无法被 Java  程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给启动类加载器，直接使用 null 代替即可。

2. **扩展类加载器（Extension ClassLoader）：**

   ​	这个类加载器是由 ExtClassLoader（sun.misc.Launcher$ExtClassLoader）实现的。它负责将<JAVA_HOME>/lib/ext 或者被 java.ext.dir  系统变量所指定路径中的所有类库加载到内存中，开发者可以直接使用扩展类加载器。

3. **系统类加载器（System ClassLoader/Application ClassLoader）：**

   ​	这个类加载器是由 AppClassLoader（sun.misc.Launcher$AppClassLoader）实现的。由于这个类加载器是  ClassLoader 中的 getSystemClassLoader()  方法的返回值，因此一般称为系统类加载器。它负责加载用户类路径（ClassPath）上所指定的类库，这些类可以在使用 -cp 或 -classpath 命令行选项调用程序时设置。开发者可以直接使用这个类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

4. **自定义加载器（User ClassLoader）：**

   ​	负责加载用户自定义路径下的类包。

## 双亲委派模型

应用程序是由三种类加载器互相配合从而实现类加载，除此之外还可以加入自己定义的类加载器。

下图展示了类加载器之间的层次关系，称为双亲委派模型（Parents Delegation  Model）。该模型要求除了顶层的启动类加载器外，其它的类加载器都要有自己的父类加载器。这里的父子关系一般通过组合关系（Composition）来实现，而不是继承关系（Inheritance）。

 [![img](https://camo.githubusercontent.com/069d7ec7d8d131fe148a3fc42eb1a27335e0aa0d/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f30646432643430612d356232622d346434352d623137362d6537356134636434626462662e706e67)](https://camo.githubusercontent.com/069d7ec7d8d131fe148a3fc42eb1a27335e0aa0d/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f30646432643430612d356232622d346434352d623137362d6537356134636434626462662e706e67) 

### 1. 工作过程

一个类加载器首先将类加载请求转发到父类加载器，只有当父类加载器无法完成时才尝试自己加载。

### 2. 好处

使得 Java 类随着它的类加载器一起具有一种带有优先级的层次关系，从而使得基础类得到统一。

例如 java.lang.Object 存放在 rt.jar 中，如果编写另外一个 java.lang.Object 并放到  ClassPath 中，程序可以编译通过。由于双亲委派模型的存在，所以在 rt.jar 中的 Object 比在 ClassPath 中的  Object 优先级更高，这是因为 rt.jar 中的 Object 使用的是启动类加载器，而 ClassPath 中的 Object  使用的是应用程序类加载器。rt.jar 中的 Object 优先级更高，那么程序中所有的 Object 都是这个 Object。

### 3. 实现

以下是抽象类 java.lang.ClassLoader 的代码片段，其中的 loadClass()  方法运行过程如下：先检查类是否已经加载过，如果没有则让父类加载器去加载。当父类加载器加载失败时抛出  ClassNotFoundException，此时尝试自己去加载。

用指定的二进制名称加载类。该方法的默认实现按以下顺序搜索类:

1. 调用findLoadedClass(字符串)检查类是否已经加载。
2. 调用父类装入器上的loadClass方法。如果父类为null，则使用虚拟机内置的类加载器（启动类加载器）。
3. 调用findClass(字符串)方法来查找类（调用子类重写的findClass方法）。

如果使用上述步骤找到类，并且解析标志为true，则此方法将在生成的类对象上调用resolveClass(类)方法。

类加载器的子类鼓励重写findClass(字符串)，而不是这个方法。

除非被覆盖，否则该方法在整个类加载过程中同步getClassLoadingLock方法的结果。

```java
public abstract class ClassLoader {
    // The parent class loader for delegation
    private final ClassLoader parent;

    public Class<?> loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }

    protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name); // 让虚拟机内置启动类加载该类
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c); // 连接该类
            }
            return c;
        }
    }

    protected Class<?> findClass(String name) throws ClassNotFoundException {
        throw new ClassNotFoundException(name);
    }
}
```

## 自定义类加载器实现

以下代码中的 FileSystemClassLoader 是自定义类加载器，继承自  java.lang.ClassLoader，用于加载文件系统上的类。它首先根据类的全名在文件系统上查找类的字节代码文件（.class  文件），然后读取该文件内容，最后通过 defineClass() 方法来把这些字节代码转换成 java.lang.Class 类的实例。

java.lang.ClassLoader 的 loadClass() 实现了双亲委派模型的逻辑，自定义类加载器一般不去重写它，但是需要重写 findClass() 方法。

```java
public class FileSystemClassLoader extends ClassLoader {

    private String rootDir;

    public FileSystemClassLoader(String rootDir) {
        this.rootDir = rootDir;
    }

    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = getClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
    }

    private byte[] getClassData(String className) {
        String path = classNameToPath(className);
        try {
            InputStream ins = new FileInputStream(path);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int bufferSize = 4096;
            byte[] buffer = new byte[bufferSize];
            int bytesNumRead;
            while ((bytesNumRead = ins.read(buffer)) != -1) {
                baos.write(buffer, 0, bytesNumRead);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    private String classNameToPath(String className) {
        return rootDir + File.separatorChar
                + className.replace('.', File.separatorChar) + ".class";
    }
}
```

**注意：**一个功能良好的类加载器应当保证以下三点属性：

- 给定相同名称的对象，类加载器应当总是返回相同的Class（类型）对象。
- 如果类加载器 *L1* 将加载类 *C* 的请求委托给另一个类加载器 *L2* ，那么对于满足下列条件之一的任意类型 *T* 来说，*L1* 和 *L2* 都应当返回相同的 Class 对象：
  - *T* 是 *C* 的直接超类或直接超接口；
  - *T* 是 *C* 中某个字段的类型；
  - *T* 是 *C* 中某个方法或构造器的形式参数类型；
  - *T* 是 *C* 中某个方法的返回值类型；
- 如果某个用户自定义的类加载器预先加载了某个类或接口的二进制表示，或批量加载了一组相关的类，并在加载时出现错误，那它就必须在程序的某个点反映出加载时的错误。而这个点，一定要和不使用预先加载或批量加载时出现错误的那个点相同。

## 类的加载时间和方式

类的加载时间有两种情况：

1. 当执行新的字节码时(例如，MyClass mc = new MyClass()）；

2. 当字节码对类(例如System.out)进行静态引用时；

   

   类加载器是分层的。第一个类是通过类（主类）中声明的静态main()方法特别加载的。所有随后加载的类都由已经加载

并正在运行的类加载。

类加载器在加载类时遵循一下规则（双亲委托机制）：

1. 检查类是否已经加载；
2. 如果未加载，委托父类加载器去加载该类；
3. 如果父类加载器无法加载该类，则尝试自身加载器加载该类；

**双亲委托机制的优势：**

- 沙箱安全机制：比如自己写的String.class类不会被加载，这样可以防止核心库被随意篡改；
- 避免类重复加载：当父加载器已经加载了该类时，就不需要子类再加载一次；

## 静态加载与动态加载

​	类是用Java的new操作符静态加载的。动态加载是一种技术，通过使用class . forname()在运行时以编程方式调用类加载器的函数。

​	类是在运行期间第一次使用时动态加载的，而不是一次性加载所有类。因为如果一次性加载，那么会占用很多的内存。

## loadClass与Class.forname之间的不同

​	**loadClass**只加载类但不初始化对象；

​	**Class.forname**会在加载类后初始化对象；

例子：如果你使用ClassLoader.loadClass去加载 JDBC 驱动类，该类不会被注册，并且程序也无法使用 JDBC。

## 类的生命周期

 [![img](https://camo.githubusercontent.com/7fcef0cbcfc984a6b4de3bb3b8a333e5b254e31c/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f33333566653139632d346137362d343561622d393332302d3838633930643661306437652e706e67)](https://camo.githubusercontent.com/7fcef0cbcfc984a6b4de3bb3b8a333e5b254e31c/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f33333566653139632d346137362d343561622d393332302d3838633930643661306437652e706e67) 

包括以下 7 个阶段：

- **加载（Loading）**
- **验证（Verification）**
- **准备（Preparation）**
- **解析（Resolution）**
- **初始化（Initialization）**
- 使用（Using）
- 卸载（Unloading）



## 类的加载过程

包含了加载、验证、准备、解析和初始化这 5 个阶段。

### 加载（Loading）

加载是类加载的一个阶段，注意不要混淆。

加载过程完成以下三件事：

- 通过类的完全限定名称获取定义该类的二进制字节流。
- 将该字节流表示的静态存储结构转换为方法区的运行时存储结构。
- 在内存中生成一个代表该类的 Class 对象，作为方法区中该类各种数据的访问入口。

其中二进制字节流可以从以下方式中获取：

- 从 ZIP 包读取，成为 JAR、EAR、WAR 格式的基础。
- 从网络中获取，最典型的应用是 Applet。
- 运行时计算生成，例如动态代理技术，在 java.lang.reflect.Proxy 使用 ProxyGenerator.generateProxyClass 的代理类的二进制字节流。
- 由其他文件生成，例如由 JSP 文件生成对应的 Class 类。

### 连接（Linking）

​	连接包括三个步骤：

- 验证：确保 Class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。

- 准备：类变量是被 static 修饰的变量（包含方法、字段和类），准备阶段为类变量分配内存并设置初始值，使用的是方法区的内存。

  实例变量不会在这阶段分配内存，它会在对象实例化时随着对象一起被分配在堆中。应该注意到，实例化不是类加载的一个过程，类加载发生在所有实例化操作之前，并且类加载只进行一次，实例化可以进行多次。

  初始值一般为 0 值，例如下面的类变量 value 被初始化为 0 而不是 123。

  ```java
  public static int value = 123;
  ```

  如果类变量是常量，那么它将初始化为表达式所定义的值而不是 0。例如下面的常量 value 被初始化为 123 而不是 0。

  ```java
  public static final int value = 123;
  ```

- 解析：将常量池的符号引用替换为直接引用的过程。

  **其中解析过程在某些情况下可以在初始化阶段之后再开始，这是为了支持 Java 的动态绑定。**

### 初始化（Initialization）

初始化阶段才真正开始执行类中定义的 Java 程序代码。初始化阶段是虚拟机执行类构造器 <clinit>() 方法的过程。在准备阶段，类变量已经赋过一次系统要求的初始值，而在初始化阶段，根据程序员通过程序制定的主观计划去初始化类变量和其它资源。

<clinit>()  是由编译器自动收集类中所有类变量的赋值动作和静态语句块中的语句合并产生的，编译器收集的顺序由语句在源文件中出现的顺序决定。特别注意的是，**静态语句块只能访问到定义在它之前的类变量，定义在它之后的类变量只能赋值，不能访问。**例如以下代码：

```java
public class Test {
    static {
        i = 0;                // 给变量赋值可以正常编译通过
        System.out.print(i);  // 这句编译器会提示“非法向前引用”
    }
    static int i = 1;
}
```

由于父类的 <clinit>() 方法先执行，也就意味着父类中定义的静态语句块的执行要优先于子类。例如以下代码：

```java
static class Parent {
    public static int A = 1;
    static {
        A = 2;
    }
}

static class Sub extends Parent {
    public static int B = A;
}

public static void main(String[] args) {
     System.out.println(Sub.B);  // 2
}
```

接口中不可以使用静态语句块，但仍然有类变量初始化的赋值操作，因此接口与类一样都会生成 <clinit>()  方法。但接口与类不同的是，执行接口的 <clinit>() 方法不需要先执行父接口的 <clinit>()  方法。只有当父接口中定义的变量使用时，父接口才会初始化。另外，接口的实现类在初始化时也一样不会执行接口的 <clinit>()  方法。

虚拟机会保证一个类的 <clinit>()  方法在多线程环境下被正确的加锁和同步，如果多个线程同时初始化一个类，只会有一个线程执行这个类的 <clinit>()  方法，其它线程都会阻塞等待，直到活动线程执行 <clinit>() 方法完毕。如果在一个类的 <clinit>()  方法中有耗时的操作，就可能造成多个线程阻塞，在实际过程中此种阻塞很隐蔽。

## 类初始化时机

### 1. 主动引用

虚拟机规范中并没有强制约束何时进行加载，但是规范严格规定了有且只有下列五种情况必须对类进行初始化（加载、验证、准备都会随之发生）：

- 遇到 new、getstatic、putstatic、invokestatic  这四条字节码指令时，如果类没有进行过初始化，则必须先触发其初始化。最常见的生成这 4 条指令的场景是：使用 new  关键字实例化对象的时候；读取或设置一个类的静态字段（**被 final  修饰、已在编译期把结果放入常量池的静态字段除外**）的时候；以及调用一个类的静态方法的时候。
- 使用 java.lang.reflect 包的方法对类进行反射调用的时候，如果类没有进行初始化，则需要先触发其初始化。
- 当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
- 当虚拟机启动时，用户需要指定一个要执行的主类（包含 main() 方法的那个类），虚拟机会先初始化这个主类；
- 当使用 JDK 1.7 的动态语言支持时，如果一个 java.lang.invoke.MethodHandle 实例最后的解析结果为  REF_getStatic, REF_putStatic, REF_invokeStatic  的方法句柄，并且这个方法句柄所对应的类没有进行过初始化，则需要先触发其初始化；

简而言之：在new、读取或设置静态字段、调用类的静态方法、反射调用、初始化类（先触发父类的初始化）、初始化主类（包含main（）方法）时，如果发现类没有初始化，则需要先初始化类。



```java
public class InitializationTest {

    static {
        System.out.println("static");
    }

    public static void main(String[] args) {
        System.out.println("main");
    }
}
//主类在虚拟机启动时初始化。
```



### 2. 被动引用

以上 5 种场景中的行为称为对一个类进行主动引用。除此之外，所有引用类的方式都不会触发初始化，称为被动引用。被动引用的常见例子包括：

- 通过子类引用父类的静态字段，不会导致子类初始化。

```java
System.out.println(SubClass.value);  // value 字段在 SuperClass 中定义
```

- 通过数组定义来引用类，不会触发此类的初始化。该过程会对数组类进行初始化，数组类是一个由虚拟机自动生成的、直接继承自 Object 的子类，其中包含了数组的属性和方法。

```java
SuperClass[] sca = new SuperClass[10];
```

- 常量在编译阶段会存入调用类的常量池中，本质上并没有直接引用到定义常量的类，因此不会触发定义常量的类的初始化。

```java
System.out.println(ConstClass.HELLOWORLD);
```

## 类相等与类加载器的关系

两个类相等，需要类本身相等，并且使用同一个类加载器进行加载。这是因为每一个类加载器都拥有一个独立的类名称空间。

这里的相等，包括类的 Class 对象的 equals() 方法、isAssignableFrom() 方法、isInstance() 方法的返回结果为 true，也包括使用 instanceof 关键字做对象所属关系判定结果为 true。

## Java虚拟机规范中的创建和加载

​     Java虚拟机启动是通过启动类加载器创建一个初始类来完成的，该类加载器是虚拟机使用c++具体实现的。接着，虚拟机连接这个初始类，初始化并调用它的静态 main 方法。之后的整个执行过程都是对此方法的调用开始的。在执行main方法中的虚拟机指令可能会导致虚拟机连接（并于其后创建）另外的一些类或接口，也可能会令虚拟机调用另外的方法。

​	如果要创建标记为 *N* 的类或接口 *C*，需要现在虚拟机方法区上为 *C* 创建与虚拟机实现相匹配的内部表示。*C* 的创建是由另一个类或接口 *D* 所出发的，它通过自己的运行时常量池引用了 *C*。*C* 的创建也可能由 *D* 调用 JavaSE平台类库中的某些方法而出发，如反射等。

​	在虚拟机运行时，类或接口是由键值对的形式确定：二进制名称和它的定义加载器共同确定的。

​	如果类加载器直接创建某个类或接口，这个类加载器被称为该类或接口的**定义加载器**。

​	如果类加载器把加载请求委托给其他的类加载器，这个类加载器被成为该类或接口的**初始加载器**。

​	**Java虚拟机通过下面三个过程之一来创建标记为 *N* 的类或接口 *C* ：**

- 如果 *N* 表示一个非数组的类或接口，那么可以用以下两种方式来加载并创建 *C* ：
  - 如果 *D* 是由启动类加载器所定义的，那么用启动类加载器来初始加载 *C* 。
  - 如果 *D* 是由用户自定义加载器所定义的，那么用该自定义加载器来初始加载 *C* 。
- 如果 *N* 表示一个数组类，那么该数组类是由虚拟机而不是类加载器创建。然而，在创建数组类 *C* 的过程中，也会用到 *D* 的定义类加载器。



# 运行时数据区

 [![img](https://camo.githubusercontent.com/397eed8a3eff96cc19cedbee1f20f100afaf6295/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f35373738643131332d386531332d346335332d623562662d3830316535383038306239372e706e67)](https://camo.githubusercontent.com/397eed8a3eff96cc19cedbee1f20f100afaf6295/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f35373738643131332d386531332d346335332d623562662d3830316535383038306239372e706e67) 



 [![img](https://camo.githubusercontent.com/397eed8a3eff96cc19cedbee1f20f100afaf6295/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f35373738643131332d386531332d346335332d623562662d3830316535383038306239372e706e67)](https://camo.githubusercontent.com/397eed8a3eff96cc19cedbee1f20f100afaf6295/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f35373738643131332d386531332d346335332d623562662d3830316535383038306239372e706e67) 



## 程序计数器

记录正在执行的虚拟机字节码指令的地址（如果正在执行的是本地方法则为空）。

## Java 虚拟机栈

每个 Java 方法在执行的同时会创建一个栈帧用于存储局部变量表、操作数栈、常量池引用等信息。从方法调用直至执行完成的过程，对应着一个栈帧在 Java 虚拟机栈中入栈和出栈的过程。

 [![img](https://camo.githubusercontent.com/96d3e63fbdedc3e6af5f8365815898bef9799ea3/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38343432353139662d306234642d343866342d383232392d3536663938343336336336392e706e67)](https://camo.githubusercontent.com/96d3e63fbdedc3e6af5f8365815898bef9799ea3/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38343432353139662d306234642d343866342d383232392d3536663938343336336336392e706e67) 



可以通过 -Xss 这个虚拟机参数来指定每个线程的 Java 虚拟机栈内存大小，在 JDK 1.4 中默认为 256K，而在 JDK 1.5+ 默认为 1M：

```
java -Xss2M HackTheJava
```

该区域可能抛出以下异常：

- 当线程请求的栈深度超过最大值，会抛出 StackOverflowError 异常；
- 栈进行动态扩展时如果无法申请到足够内存，会抛出 OutOfMemoryError 异常。



## 本地方法栈

本地方法栈与 Java 虚拟机栈类似，它们之间的区别只不过是本地方法栈为本地方法服务。

本地方法一般是用其它语言（C、C++ 或汇编语言等）编写的，并且被编译为基于本机硬件和操作系统的程序，对待这些方法需要特别处理。

 [![img](https://camo.githubusercontent.com/1d8a26c9cee39aa0d6f7e1a165b422f0fa0d4187/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f36366136383939642d633662302d346134372d383536392d3964303866306261663836632e706e67)](https://camo.githubusercontent.com/1d8a26c9cee39aa0d6f7e1a165b422f0fa0d4187/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f36366136383939642d633662302d346134372d383536392d3964303866306261663836632e706e67) 





## 堆

所有对象都在这里分配内存，是垃圾收集的主要区域（"GC 堆"）。

现代的垃圾收集器基本都是采用分代收集算法，其主要的思想是针对不同类型的对象采取不同的垃圾回收算法。可以将堆分成两块：

- 新生代（Young Generation）
- 老年代（Old Generation）

堆不需要连续内存，并且可以动态增加其内存，增加失败会抛出 OutOfMemoryError 异常。

可以通过 -Xms 和 -Xmx 这两个虚拟机参数来指定一个程序的堆内存大小，第一个参数设置初始值，第二个参数设置最大值。

```
java -Xms1M -Xmx2M HackTheJava
```



## 方法区

用于存放已被加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

和堆一样不需要连续的内存，并且可以动态扩展，动态扩展失败一样会抛出 OutOfMemoryError 异常。

对这块区域进行垃圾回收的主要目标是对常量池的回收和对类的卸载，但是一般比较难实现。

HotSpot 虚拟机把它当成永久代来进行垃圾回收。但很难确定永久代的大小，因为它受到很多因素影响，并且每次 Full GC  之后永久代的大小都会改变，所以经常会抛出 OutOfMemoryError 异常。为了更容易管理方法区，从 JDK 1.8  开始，移除永久代，并把方法区移至元空间，它位于本地内存中，而不是虚拟机内存中。

方法区是一个 JVM 规范，永久代与元空间都是其一种实现方式。在 JDK 1.8 之后，原来永久代的数据被分到了堆和元空间中。元空间存储类的元信息，静态变量和常量池等放入堆中。



## 运行时常量池

运行时常量池是方法区的一部分。

Class 文件中的常量池（编译器生成的字面量和符号引用）会在类加载后被放入这个区域。

除了在编译期生成的常量，还允许动态生成，例如 String 类的 intern()。



## 直接内存

在 JDK 1.4 中新引入了 NIO 类，它可以使用 Native 函数库直接分配堆外内存，然后通过 Java 堆里的  DirectByteBuffer 对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在堆内存和堆外内存来回拷贝数据。

# 执行引擎



# 本地方法接口（JNI）



# 参考资料

[Java Class Loaders Explained]: https://javatutorial.net/java-class-loaders-explained

