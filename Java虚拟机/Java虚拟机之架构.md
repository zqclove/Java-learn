[TOC]

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



# 参考资料

[Java Class Loaders Explained](https://javatutorial.net/java-class-loaders-explained)

[CyC2018/CS-Notes/java 虚拟机](https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20%E8%99%9A%E6%8B%9F%E6%9C%BA.md)

 **深入理解Java虚拟机：JVM高级特性与最佳实践（第3版） (华章原创精品) - 周志明**