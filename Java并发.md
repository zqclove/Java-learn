[TOC]



## Java并发

## 一、线程的使用

三种使用线程的方法：

- 实现Runnable接口；
- 实现Callable接口；
- 继承Thread类；

实现Runnable接口和Callable接口的类只能当做一个在线程中运行的任务，不是真正意义上的线程，因此最后还是需要通过Thread来调用。可以理解为任务是通过线程驱动从而执行的。

### Runnable

需要实现接口中的 run() 方法。

```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        // ...
    }
}
```

使用 Runnable 实例再创建一个 Thread 实例，然后调用 Thread 实例的 start() 方法来启动线程。

```java
public static void main(String[] args) {
    MyRunnable instance = new MyRunnable();
    Thread thread = new Thread(instance);
    thread.start();
}
```

### Calleable

与 Runnable 相比，Callable 可以有返回值，返回值通过 FutureTask 进行封装。

```java
public class MyCallable implements Callable<Integer> {
    public Integer call() {
        return 123;
    }
}
```

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    MyCallable mc = new MyCallable();
    FutureTask<Integer> ft = new FutureTask<>(mc);
    Thread thread = new Thread(ft);
    thread.start();
    System.out.println(ft.get());
}
```

### 继承 Thread 类

同样也是需要实现 run() 方法，因为 Thread 类也实现了 Runable 接口。

当调用 start() 方法启动一个线程时，虚拟机会将该线程放入就绪队列中等待被调度，当一个线程被调度时会执行该线程的 run() 方法。

```java
public class MyThread extends Thread {
    public void run() {
        // ...
    }
    public static void main(String[] args) {
   	 	MyThread mt = new MyThread();
    	mt.start();
	}
}
```

### 实现接口 VS 继承 Thread

实现接口会更好一些，因为：

- Java 不支持多继承，因此继承了 Thread 类就无法继承其它类，但是可以实现多个接口；
- 类可能只要求可执行就行，继承整个 Thread 类开销过大。

## 二、线程机制

### Executor

Executor 管理多个异步任务的执行，而无需程序员显式地管理线程的生命周期。这里的异步是指多个任务的执行互不干扰，不需要进行同步操作。

主要有三种 Executor：

- CachedThreadPool：一个任务创建一个线程；
- FixedThreadPool：所有任务只能使用固定大小的线程；
- SingleThreadExecutor：相当于大小为 1 的 FixedThreadPool。

```java
public static void main(String[] args) {
    ExecutorService executorService = Executors.newCachedThreadPool();
    for (int i = 0; i < 5; i++) {
        executorService.execute(new MyRunnable());
    }
    executorService.shutdown();
}
```

### Daemon

守护线程是程序运行时在后台提供服务的线程，不属于程序中不可或缺的部分。

当所有非守护线程结束时，程序也就终止，同时会杀死所有守护线程。

main() 属于非守护线程，GC属于守护线程。

在线程启动之前使用 setDaemon() 方法可以将一个线程设置为守护线程。

```java
public static void main(String[] args) {
    Thread thread = new Thread(new MyRunnable());
    thread.setDaemon(true);
}
```

### sleep()

Thread.sleep(millisec) 方法会休眠当前正在执行的线程，millisec 单位为毫秒。

**sleep() 可能会抛出 InterruptedException**，因为异常不能跨线程传播回 main() 中，因此必须在本地进行处理。线程中抛出的其它异常也同样需要在本地进行处理。

```java
public void run() {
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

### yield()

对静态方法 Thread.yield() 的调用声明了当前线程已经完成了生命周期中最重要的部分，可以切换给其它线程来执行。该方法只是对线程调度器的一个建议，而且也只是建议具有相同优先级的其它线程可以运行。

```java
public void run() {
    Thread.yield();
}
```

### 关于创建线程池的开发经验

#### ThreadPoolExecutor替代Executors

线程池**不允许**使用 Executors 去创建，而是通过 **ThreadPoolExecutor** 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。
说明： Executors 返回的线程池对象的弊端如下：
1） FixedThreadPool 和 SingleThreadPool :
​	允许的**请求队列**长度为 Integer.MAX_VALUE ，可能会堆积大量的请求，从而导致 OOM 。
2） CachedThreadPool 和 ScheduledThreadPool :
​	允许的**创建线程**数量为 Integer.MAX_VALUE ，可能会创建大量的线程，从而导致 OOM 。

这四种创建线程池的方式都是传递不同参数、不同策略给ThreadPoolExecutor去创建线程池，即预定义的创建线程池方式。

```java
	public ThreadPoolExecutor(int corePoolSize,	// 核心线程池大小
                              int maximumPoolSize, // 最大线程池大小
                              long keepAliveTime, // 线程最大空闲时间
                              TimeUnit unit, // 时间单元（增加代码可读性）
                              BlockingQueue<Runnable> workQueue, // 线程等待队列
                              ThreadFactory threadFactory, // 线程创建工厂
                              RejectedExecutionHandler handler) { // 拒绝策略
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

各参数文档解释：

- corePoolSize：池中保留的线程数，即使它们是空闲的，除非设置了allowCoreThreadTimeOut。
- maximumPoolSize：池中允许的最大线程数。
- keepAliveTime：当线程数量大于核心线程数时，这是多余（临时）空闲线程在终止之前等待新任务的最大时间。（将转换成纳秒单位）
- unit：keepAliveTime参数的时间单位。
- workQueue：在执行任务之前用于保存任务的队列。此队列将只保存由execute方法提交的可运行任务。
- threadFactory：执行程序创建新线程时要使用的工厂。
- handler：当执行因达到线程边界和队列容量而阻塞时使用的处理程序。

当一个任务通过execute(Runnable)方法欲添加到线程池时：

1.如果线程池中线程数量小于corePoolSize，即使线程中的线程都处于空闲状态，也要创建新的线程来处理被添加的任务。

2.如果线程池中线程数量等于corePoolSize，但是workQueue未满，那么任务被放入workQueue中。

3.如果线程池中线程数量大于corePoolSize，即workQueue满（workQueue是有界队列），则判断线程数量是否小于maximumPoolSize，小于则创建新的线程来处理对应的任务（处理完后立即退出），大于则进入第4步。

4.此时线程池等于maximumPoolSize，则按照给定的拒绝策略拒绝任务。

#### 线程资源必须通过线程池提供

线程资源必须通过线程池提供，不允许在应用中自行显式创建线程。
使用线程池的好处是减少在创建和销毁线程上所消耗的时间以及系统资源的开销，解决资源不足的问题。如果不使用线程池，有可能造成系统创建大量同类线程而导致消耗完内存或者“过度切换”的问题。

### 场景模拟

**FixedThreadPool-线程池大小固定，任务队列无界**

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

FixedThreadPool的优点是能够保证所有的任务都被执行，永远不会拒绝新的任务；同时缺点是队列数量没有限制，在任务执行时间无限延长的这种极端情况下会造成内存问题。

适用场景：可用于Web服务瞬时削峰，但需注意长时间持续高峰情况造成的队列阻塞。

**SingleThreadExecutor-线程池大小固定为1，任务队列无边界**

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

这个工厂方法中使用无界LinkedBlockingQueue，并的将线程数设置成1，除此以外还使用FinalizableDelegatedExecutorService类进行了包装。这个包装类的主要目的是为了屏蔽ThreadPoolExecutor中动态修改线程数量的功能，仅保留ExecutorService中提供的方法。虽然是单线程处理，一旦线程因为处理异常等原因终止的时候，ThreadPoolExecutor会自动创建一个新的线程继续进行工作。

```java
	public static void main(String[] args) {
        ExecutorService fixedExecutorService = Executors.newFixedThreadPool(1);
        ThreadPoolExecutor threadPoolExecutor 
       	     = (ThreadPoolExecutor) fixedExecutorService;
        System.out.println(threadPoolExecutor.getMaximumPoolSize());
        threadPoolExecutor.setCorePoolSize(8); // 动态修改线程池数量
        
        ExecutorService singleExecutorService = Executors.newSingleThreadExecutor();
//      运行时异常 java.lang.ClassCastException
//      ThreadPoolExecutor threadPoolExecutor2 = (ThreadPoolExecutor) singleExecutorService;
    }

```

SingleThreadExecutor 适用于在逻辑上需要单线程处理任务的场景，同时无界的LinkedBlockingQueue保证新任务都能够放入队列，不会被拒绝；缺点和FixedThreadPool相同，当处理任务无限等待的时候会造成内存问题。

**CachedThreadPool-线程池无限大（Integer.MAX_VALUE），等待队列长度为1**

```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

SynchronousQueue是一个只有1个元素的队列，入队的任务需要一直等待直到队列中的元素被移出。核心线程数是0，意味着所有任务会先入队列；最大线程数是Integer.MAX_VALUE，可以认为线程数量是没有限制的。KeepAlive时间被设置成60秒，意味着在没有任务的时候线程等待60秒以后退出。CachedThreadPool对任务的处理策略是提交的任务会立即分配一个线程进行执行，线程池中线程数量会随着任务数的变化自动扩张和缩减，在任务执行时间无限延长的极端情况下会创建过多的线程。

适用场景：快速处理大量耗时较短的任务，如Netty的NIO接受请求时，可使用CachedThreadPool。

## 三、中断机制

一个线程执行完毕之后会自动结束，如果在运行过程中发生异常也会提前结束。

### InterruptedException

通过调用一个线程的 interrupt() 来中断该线程，如果该线程处于阻塞、限期等待或者无限期等待状态，那么就会抛出 InterruptedException，从而提前结束该线程。但是不能中断 I/O 阻塞和 synchronized 锁阻塞。

对于以下代码，在 main() 中启动一个线程之后再中断它，由于线程中调用了 Thread.sleep() 方法，因此会抛出一个 InterruptedException，从而提前结束线程，不执行之后的语句。

```java
public class InterruptExample {

    private static class MyThread1 extends Thread {
        @Override
        public void run() {
            try {
                Thread.sleep(2000);
                System.out.println("Thread run");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
public static void main(String[] args) throws InterruptedException {
    Thread thread1 = new MyThread1();
    thread1.start();
    thread1.interrupt();
    System.out.println("Main run");
}
```

```java
Main run
java.lang.InterruptedException: sleep interrupted
    at java.lang.Thread.sleep(Native Method)
    at InterruptExample.lambda$main$0(InterruptExample.java:5)
    at InterruptExample$$Lambda$1/713338599.run(Unknown Source)
    at java.lang.Thread.run(Thread.java:745)
```

### interrupted()

如果一个线程的 run() 方法执行一个无限循环，并且没有执行 sleep() 等会抛出 InterruptedException 的操作，那么调用线程的 interrupt() 方法就无法使线程提前结束。

但是调用 interrupt() 方法会设置线程的中断标记，此时调用 interrupted() 方法会返回 true。因此可以在循环体中使用 interrupted() 方法来判断线程是否处于中断状态，从而提前结束线程。

```java
public class InterruptExample {

    private static class MyThread2 extends Thread {
        @Override
        public void run() {
            while (!interrupted()) {
                // ..
            }
            System.out.println("Thread end");
        }
    }
}
```

```java
public static void main(String[] args) throws InterruptedException {
    Thread thread2 = new MyThread2();
    thread2.start();
    thread2.interrupt();
}
```

```java
Thread end
```

### Executor 的中断操作

调用 Executor 的 shutdown() 方法会等待线程都执行完毕之后再关闭，但是如果调用的是 shutdownNow() 方法，则相当于调用每个线程的 interrupt() 方法。

以下使用 Lambda 创建线程，相当于创建了一个匿名内部线程。

```java
public static void main(String[] args) {
    ExecutorService executorService = Executors.newCachedThreadPool();
    executorService.execute(() -> {
        try {
            Thread.sleep(2000);
            System.out.println("Thread run");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
    executorService.shutdownNow();
    System.out.println("Main run");
}
```

```java
Main run
java.lang.InterruptedException: sleep interrupted
    at java.lang.Thread.sleep(Native Method)
    at ExecutorInterruptExample.lambda$main$0(ExecutorInterruptExample.java:9)
    at ExecutorInterruptExample$$Lambda$1/1160460865.run(Unknown Source)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
```

如果只想中断 Executor 中的一个线程，可以通过使用 submit() 方法来提交一个线程，它会返回一个 Future<?> 对象，通过调用该对象的 cancel(true) 方法就可以中断线程。

```java
Future<?> future = executorService.submit(() -> {
    // ..
});	
future.cancel(true);
```

## 四、互斥同步

Java 提供了两种锁机制来控制多个线程对共享资源的互斥访问，第一个是 JVM 实现的 synchronized，而另一个是 JDK 实现的 ReentrantLock。

### synchronized

#### **1. 同步一个代码块**

```java
public void func() {
    synchronized (this) {
        // ...
    }
}
```

**它只作用于同一个对象，如果调用两个对象上的同步代码块，就不会进行同步。**

对于以下代码，使用 ExecutorService 执行了两个线程，由于调用的是同一个对象的同步代码块，因此这两个线程会进行同步，当一个线程进入同步语句块时，另一个线程就必须等待。

```java
public class SynchronizedExample {

    public void func1() {
        synchronized (this) {
            for (int i = 0; i < 10; i++) {
                System.out.print(i + " ");
            }
        }
    }
    public static void main(String[] args) {
        SynchronizedExample e1 = new SynchronizedExample();
        ExecutorService executorService = Executors.newCachedThreadPool();
   	    executorService.execute(() -> e1.func1());
  	  	executorService.execute(() -> e1.func1());
	}
}
//0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9
```

对于以下代码，两个线程调用了不同对象的同步代码块，因此这两个线程就不需要同步。从输出结果可以看出，两个线程交叉执行。

```java
public static void main(String[] args) {
    SynchronizedExample e1 = new SynchronizedExample();
    SynchronizedExample e2 = new SynchronizedExample();
    ExecutorService executorService = Executors.newCachedThreadPool();
    executorService.execute(() -> e1.func1());
    executorService.execute(() -> e2.func1());
}
//0 0 1 1 2 2 3 3 4 4 5 5 6 6 7 7 8 8 9 9
```

#### **2. 同步一个方法**

```java
public synchronized void func () {
    // ...
}
```

它和同步代码块一样，作用于同一个对象。

#### **3. 同步一个类**

```java
public void func() {
    synchronized (SynchronizedExample.class) {
        // ...
    }
}
```

作用于整个类，也就是说两个线程调用**同一个类的不同对象**上的这种同步语句，也会进行同步。

```java
public class SynchronizedExample {

    public void func2() {
        synchronized (SynchronizedExample.class) {
            for (int i = 0; i < 10; i++) {
                System.out.print(i + " ");
            }
        }
    }
    public static void main(String[] args) {
    	SynchronizedExample e1 = new SynchronizedExample();
    	SynchronizedExample e2 = new SynchronizedExample();
    	ExecutorService executorService = Executors.newCachedThreadPool();
    	executorService.execute(() -> e1.func2());
    	executorService.execute(() -> e2.func2());
	}
}
//0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9
```

#### **4. 同步一个静态方法**

```java
public synchronized static void fun() {
    // ...
}
```

作用于整个类。

#### 具有可重入性

​	若一个程序或子程序可以“在任意时刻被中断然后操作系统调度执行另外一段代码，这段代码又调用了该子程序不会出错”，则称其为可重入（reentrant或re-entrant）的。即当该子程序正在运行时，执行线程可以再次进入并执行它，仍然获得符合设计时预期的结果。与多线程并发执行的线程安全不同，可重入强调对单个线程执行时重新进入同一个子程序仍然是安全的。

代码示例：

```java

public class Xttblog extends SuperXttblog {
    public static void main(String[] args) {
        Xttblog child = new Xttblog();
        child.doSomething();
    }
 
    public synchronized void doSomething() {
        System.out.println("child.doSomething()" + Thread.currentThread().getName());
        doAnotherThing(); // 调用自己类中其他的synchronized方法
    }
 
    private synchronized void doAnotherThing() {
        super.doSomething(); // 调用父类的synchronized方法
        System.out.println("child.doAnotherThing()" + Thread.currentThread().getName());
    }
}
 
class SuperXttblog {
    public synchronized void doSomething() {
        System.out.println("father.doSomething()" + Thread.currentThread().getName());
    }
}
//child.doSomething()Thread-5492
//father.doSomething()Thread-5492
//child.doAnotherThing()Thread-5492
```

​	在同一个线程中，一个同步方法获取可重入锁后调用另一个同步方法时，如果该锁是同一个锁对象，则计数器+1，获取锁成功，直接执行同步方法。在释放可重入锁时，计数器-1，直到0则释放锁，其他线程才可以获取该锁对象。

### ReentrantLock

可重入锁：当线程请求一个由其它线程持有的对象锁时，该线程会阻塞，而当线程请求由自己持有的对象锁时，如果该锁是重入锁，请求就会成功，否则阻塞。synchronized 也是可重入锁。

ReentrantLock 是 java.util.concurrent（J.U.C）包中的锁。

```java
public class LockExample {

    private Lock lock = new ReentrantLock();

    public void func() {
        lock.lock();
        try {
            for (int i = 0; i < 10; i++) {
                System.out.print(i + " ");
            }
        } finally {
            lock.unlock(); // 确保释放锁，从而避免发生死锁。
        }
    }
    public static void main(String[] args) {
    	LockExample lockExample = new LockExample();
    	ExecutorService executorService = Executors.newCachedThreadPool();
    	executorService.execute(() -> lockExample.func());
    	executorService.execute(() -> lockExample.func());
	}
}
//0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9
```

### 比较

#### **1. 锁的实现**

synchronized 是 JVM 实现的，而 ReentrantLock 是 JDK 实现的。

#### **2. 性能**

新版本 Java 对 synchronized 进行了很多优化，例如自旋锁等，synchronized 与 ReentrantLock 大致相同。

#### **3. 等待可中断**

当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情。

ReentrantLock 可中断，而 synchronized 不行。

#### **4. 公平锁**

公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁。

synchronized 中的锁是非公平的，ReentrantLock 默认情况下也是非公平的，但是也可以是公平的。

#### **5. 锁绑定多个条件**

一个 ReentrantLock 可以同时绑定多个 Condition 对象。

### 使用选择

除非需要使用 ReentrantLock 的高级功能，否则优先使用 synchronized。这是因为 synchronized 是  JVM 实现的一种锁机制，JVM 原生地支持它，而 ReentrantLock 不是所有的 JDK 版本都支持。并且使用  synchronized 不用担心没有释放锁而导致死锁问题，因为 JVM 会确保锁的释放。

### 可重入锁理解

ReentrantLock翻译过来为可重入锁，它的可重入性表现在同一个线程可以多次获得锁，而不同线程依然不可多次获得锁。ReentrantLock分为公平锁和非公平锁，公平锁保证等待时间最长的线程将优先获得锁，而非公平锁并不会保证多个线程获得锁的顺序，但是非公平锁的并发性能表现更好，ReentrantLock默认使用非公平锁。

## 五、线程之间的协作

当多个线程可以一起工作去解决某个问题时，如果某些部分必须在其它部分之前完成，那么就需要对线程进行协调。

### join()

在线程中调用另一个线程的 join() 方法，会将当前线程挂起，而不是忙等待，直到目标线程结束。

对于以下代码，虽然 b 线程先启动，但是因为在 b 线程中调用了 a 线程的 join() 方法，b 线程会等待 a 线程结束才继续执行，因此最后能够保证 a 线程的输出先于 b 线程的输出。

```java
public class JoinExample {

    private class A extends Thread {
        @Override
        public void run() {
            System.out.println("A");
        }
    }

    private class B extends Thread {

        private A a;

        B(A a) {
            this.a = a;
        }

        @Override
        public void run() {
            try {
                a.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("B");
        }
    }

    public void test() {
        A a = new A();
        B b = new B(a);
        b.start();
        a.start();
    }
    public static void main(String[] args) {
    	JoinExample example = new JoinExample();
    	example.test();
	}
}
//A
//B
```

### wait() notify() notifyAll()

调用 wait() 使得线程等待某个条件满足，线程在等待时会被**挂起**，当其他线程的运行使得这个条件满足时，其它线程会调用 notify() 或者 notifyAll() 来**唤醒挂起的线程**。

它们都属于 Object 的一部分，而不属于 Thread。

只能用在同步方法或者同步控制块中使用，否则会在运行时抛出 IllegalMonitorStateException。

使用 **wait() 挂起期间，线程会释放锁**。这是因为，如果没有释放锁，那么其它线程就无法进入对象的同步方法或者同步控制块中，那么就无法执行 notify() 或者 notifyAll() 来唤醒挂起的线程，造成死锁。

```java
public class WaitNotifyExample {

    public synchronized void before() {
        System.out.println("before");
        notifyAll();
    }

    public synchronized void after() {
        try {
            wait();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("after");
    }
    public static void main(String[] args) {
    	ExecutorService executorService = Executors.newCachedThreadPool();
    	WaitNotifyExample example = new WaitNotifyExample();
    	executorService.execute(() -> example.after());
    	executorService.execute(() -> example.before());
	}
}
before
after
```

**wait() 和 sleep() 的区别**

- wait() 是 Object 的方法，而 sleep() 是 Thread 的静态方法；
- wait() 会释放锁，sleep() 不会。

### await() signal() signalAll()

java.util.concurrent 类库中提供了 Condition 类来实现线程之间的协调，可以在 Condition 上调用 await() 方法使线程等待，其它线程调用 signal() 或 signalAll() 方法唤醒等待的线程。

相比于 wait() 这种等待方式，await() 可以指定等待的条件，因此更加灵活。

使用 Lock 来获取一个 Condition 对象。

```java
public class AwaitSignalExample {

    private Lock lock = new ReentrantLock();
    private Condition acondition = lock.newCondition(); // 可以指定等待的条件，更加灵活
	private Condition bcondition = lock.newCondition();

    public void signalX() {
        lock.lock();
        try {
            System.out.println("signalX");
			acondition.signal();         //唤醒等待acondition的线程，执行await()后面的代码
			bcondition.signal();		 //唤醒等待bcondition的线程，执行await()后面的代码
			System.out.println("after signal");
//			acondition.signalAll();
        } finally {
            lock.unlock();
        }
    }

    private void afunc() {
		lock.lock();
		try {
			acondition.await();
			System.out.println("A");
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}

	}

	private void bfunc() {
		lock.lock();
		try {
			bcondition.await();
			System.out.println("B");
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}

	}

	public static void main(String[] args) {
		AwatiAndSignal aas = new AwatiAndSignal();
		ExecutorService executorService = Executors.newCachedThreadPool();
		executorService.execute(() -> aas.afunc());
		executorService.execute(() -> aas.bfunc());
		executorService.execute(() -> aas.signalX());
		
		executorService.shutdown();
	}
}
```

**遇到的问题**

1.根据如上代码，有时两个func（）都能执行await（）后面的代码，有时只能是其中一个。

2.如果执行signalX（）方法的线程先获取到锁（并且只有一个），那么func（）方法将永远不能被唤醒。而使用Object的wait（）和notify（）则不出现此状况。

**原因分析**

1.afunc（）、bfunc（）和signalx（）三个线程在争取锁的顺序决定最后输出结果。不同的执行顺序决定不同的结果。

2.可能是signalX（）和func（）方法是普通方法，在方法内进行锁操作，而Object的例子是同步方法的方式。

### 知识扩展

#### **wait（）方法相关**

当前线程必须拥有该对象的监视器（锁机制中的monitor）。该线程释放该监视器的所有权，并等待另一个线程通

过调用notify方法或notifyAll方法通知等待该对象监视器的线程。然后线程等待，直到它可以重新获得监视器的所

有权并继续执行。

线程在调用wait（）时，

```java
public final void wait() throws InterruptedException {
    wait(0);
}
```

wait（）方法调用本地方法wait（long），该方法可以给线程添加等待时间，当达到等待时间后重新获得监视器的所有权并继续执行。参数为0则不设置等待时间。

此方法导致当前线程(称线程T)将自己置于该对象的等待集中，然后放弃该对象上的任何和所有同步声明。为了线程调度的目的，线程T被禁用，处于休眠状态，直到发生以下四件事之一:

- 其他一些线程调用这个对象的notify方法，线程T正好被任意选择为要被唤醒的线程。
- 其他一些线程调用此对象的notifyAll方法。

- 其他一些线程中断线程T。
- 指定的等待时间已到。但是，如果超时为零，则不考虑时间，线程只是等待，直到得到通知。

#### **notify（）和notifyAll（）**

notify（）和notifyAll（）都作用于对象级别上的锁，不是同一对象将不会被唤醒。

**notifyAll（）：**唤醒在该对象监视器上等待的所有线程。线程通过调用其中一个等待方法来等待对象的监视器。

此方法只能由该对象监视器的所有者线程调用。有关线程成为监视器所有者的方法，请参阅notify方法。

**notify（）：**唤醒在该对象监视器上等待的单个线程。如果有多个线程在等待该对象，则选择其中一个被唤醒。**选择是任意的，由实现自行决定**。线程通过调用其中一个等待方法来等待对象的监视器。

此方法只能由该对象监视器的所有者线程调用。线程通过以下三种方式之一成为对象监视器的所有者:

- 通过执行该对象的同步实例方法。

- 通过执行在对象上同步的同步语句的主体。

- 对于类类型的对象，通过执行该类的同步静态方法。

一次只能有一个线程拥有对象的监视器。

**共同点：**被唤醒的线程将无法继续操作，直到当前线程放弃该对象上的锁。**被唤醒的线程将以通常的方式与其他可能在此对象上积极竞争同步的线程竞争**;例如，被唤醒的线程在成为下一个锁定该对象的线程时没有可靠的特权或不利因素。

## 六、线程状态

一个线程只能处于一种状态，并且这里的线程状态特指 Java 虚拟机的线程状态，不能反映线程在特定操作系统下的状态。

### 新建（NEW）

创建后尚未启动。

### 可运行（RUNABLE）

正在 Java 虚拟机中运行。但是在操作系统层面，它可能处于运行状态，也可能等待资源调度（例如处理器资源），资源调度完成就进入运行状态。所以该状态的可运行是指可以被运行，具体有没有运行要看底层操作系统的资源调度。

### 阻塞（BLOCKED）

请求获取 monitor lock 从而进入 synchronized 函数或者代码块，但是其它线程已经占用了该 monitor lock，所以处于阻塞状态。要结束该状态进入从而 RUNABLE 需要其他线程释放 monitor lock。

### 无限期等待（WAITING）

等待其它线程显式地唤醒。

阻塞和等待的区别在于，阻塞是被动的，它是在等待获取 monitor lock。而等待是主动的，通过调用  Object.wait() 等方法进入。

| 进入方法                                   | 退出方法                             |
| ------------------------------------------ | ------------------------------------ |
| 没有设置 Timeout 参数的 Object.wait() 方法 | Object.notify() / Object.notifyAll() |
| 没有设置 Timeout 参数的 Thread.join() 方法 | 被调用的线程执行完毕                 |
| LockSupport.park() 方法                    | LockSupport.unpark(Thread)           |

### 限期等待（TIMED_WAITING）

无需等待其它线程显式地唤醒，在一定时间之后会被系统自动唤醒。

| 进入方法                                 | 退出方法                                        |
| ---------------------------------------- | ----------------------------------------------- |
| Thread.sleep() 方法                      | 时间结束                                        |
| 设置了 Timeout 参数的 Object.wait() 方法 | 时间结束 / Object.notify() / Object.notifyAll() |
| 设置了 Timeout 参数的 Thread.join() 方法 | 时间结束 / 被调用的线程执行完毕                 |
| LockSupport.parkNanos() 方法             | LockSupport.unpark(Thread)                      |
| LockSupport.parkUntil() 方法             | LockSupport.unpark(Thread)                      |

调用 Thread.sleep() 方法使线程进入限期等待状态时，常常用“使一个线程睡眠”进行描述。调用 Object.wait()  方法使线程进入限期等待或者无限期等待时，常常用“挂起一个线程”进行描述。睡眠和挂起是用来描述行为，而阻塞和等待用来描述状态。

### 死亡（TERMINATED）

可以是线程结束任务之后自己结束，或者产生了异常而结束。

[Java SE 9 Enum Thread.State](https://docs.oracle.com/javase/9/docs/api/java/lang/Thread.State.html)

## 七、J.U.C

java.util.concurrent（J.U.C）大大提高了并发性能，AQS 被认为是 J.U.C 的核心。

### CountDownLatch

**用来控制一个或者多个线程等待多个线程。**

维护了一个计数器 cnt，每次调用 countDown() 方法会让计数器的值减 1，减到 0 的时候，那些因为调用 await() 方法而在等待的线程就会被唤醒。

 [![img](https://camo.githubusercontent.com/c2a94b75d7379c204996f24411a9c497125cfa06/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f62613037383239312d373931652d343337382d623664312d6563653736633266306231342e706e67)](https://camo.githubusercontent.com/c2a94b75d7379c204996f24411a9c497125cfa06/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f62613037383239312d373931652d343337382d623664312d6563653736633266306231342e706e67) 



```java
public class CountdownLatchExample {

    public static void main(String[] args) throws InterruptedException {
        final int totalThread = 10;
        CountDownLatch countDownLatch = new CountDownLatch(totalThread);
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < totalThread; i++) {
            executorService.execute(() -> {
                System.out.print("run..");
                countDownLatch.countDown();
            });
        }
        countDownLatch.await();
        System.out.println("end");
        executorService.shutdown();
    }
}
//run..run..run..run..run..run..run..run..run..run..end
```

### CyclicBarrier

**用来控制多个线程互相等待，只有当多个线程都到达时，这些线程才会继续执行。**

和 CountdownLatch 相似，都是通过维护计数器来实现的。线程执行 await() 方法之后计数器会减 1，并进行等待，直到计数器为 0，所有调用 await() 方法而在等待的线程才能继续执行。

CyclicBarrier 和 CountdownLatch 的一个区别是，CyclicBarrier 的计数器通过调用 reset() 方法可以循环使用，所以它才叫做循环屏障。

CyclicBarrier 有两个构造函数，其中 parties 指示计数器的初始值，barrierAction 在所有线程都到达屏障的时候会执行一次。

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}

public CyclicBarrier(int parties) {
    this(parties, null);
}
```

 [![img](https://camo.githubusercontent.com/10dd07a9c7828fab8f68a0f953755869dc286a8e/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f66373161663636622d306435342d343339392d613434622d6634376235383332313938342e706e67)](https://camo.githubusercontent.com/10dd07a9c7828fab8f68a0f953755869dc286a8e/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f66373161663636622d306435342d343339392d613434622d6634376235383332313938342e706e67) 



```java
public class CyclicBarrierExample {

    public static void main(String[] args) {
        final int totalThread = 10;
        CyclicBarrier cyclicBarrier = new CyclicBarrier(totalThread);
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < totalThread; i++) {
            executorService.execute(() -> {
                System.out.print("before..");
                try {
                    cyclicBarrier.await();
                } catch (InterruptedException | BrokenBarrierException e) {
                    e.printStackTrace();
                }
                System.out.print("after..");
            });
        }
        executorService.shutdown();
    }
}
//before..before..before..before..before..before..before..before..before..before..after..after..after..after..after..after..after..after..after..after..
```

### Semaphore

**Semaphore 类似于操作系统中的信号量，可以控制对互斥资源的访问线程数。**

以下代码模拟了对某个服务的并发请求，每次只能有 3 个客户端同时访问，请求总数为 10。

```java
public class SemaphoreExample {

    public static void main(String[] args) {
        final int clientCount = 3;
        final int totalRequestCount = 10;
        Semaphore semaphore = new Semaphore(clientCount);
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < totalRequestCount; i++) {
            executorService.execute(()->{
                try {
                    semaphore.acquire();
                    System.out.print(semaphore.availablePermits() + " ");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release();
                }
            });
        }
        executorService.shutdown();
    }
}
//2 1 2 2 2 2 2 1 2 2
```

### J.U.C - 其它组件

#### FutureTask

在介绍 Callable 时我们知道它可以有返回值，返回值通过 Future 进行封装。FutureTask 实现了  RunnableFuture 接口，该接口继承自 Runnable 和 Future 接口，这使得 FutureTask  既可以当做一个任务执行，也可以有返回值。

```java
public class FutureTask<V> implements RunnableFuture<V>
public interface RunnableFuture<V> extends Runnable, Future<V>
```

**FutureTask 可用于异步获取执行结果或取消执行任务的场景。**当一个计算任务需要执行很长时间，那么就可以用 FutureTask 来封装这个任务，主线程在完成自己的任务之后再去获取结果。

```java
public class FutureTaskExample {

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<Integer> futureTask = new FutureTask<Integer>(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                int result = 0;
                for (int i = 0; i < 100; i++) {
                    Thread.sleep(10);
                    result += i;
                }
                return result;
            }
        });

        Thread computeThread = new Thread(futureTask);
        computeThread.start();

        Thread otherThread = new Thread(() -> {
            System.out.println("other task is running...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        otherThread.start();
        System.out.println(futureTask.get());
    }
}
//other task is running...
//4950
```

#### BlockingQueue

java.util.concurrent.BlockingQueue 接口有以下阻塞队列的实现：

- **FIFO 队列**  ：LinkedBlockingQueue、ArrayBlockingQueue（固定长度）
- **优先级队列**  ：PriorityBlockingQueue

提供了阻塞的 take() 和 put() 方法：如果队列为空 take() 将阻塞，直到队列中有内容；如果队列为满 put() 将阻塞，直到队列有空闲位置。

**使用 BlockingQueue 实现生产者消费者问题**

```java
public class ProducerConsumer {

    private static BlockingQueue<String> queue = new ArrayBlockingQueue<>(5);

    private static class Producer extends Thread {
        @Override
        public void run() {
            try {
                queue.put("product");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.print("produce..");
        }
    }

    private static class Consumer extends Thread {

        @Override
        public void run() {
            try {
                String product = queue.take();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.print("consume..");
        }
    }
}
public static void main(String[] args) {
    for (int i = 0; i < 2; i++) {
        Producer producer = new Producer();
        producer.start();
    }
    for (int i = 0; i < 5; i++) {
        Consumer consumer = new Consumer();
        consumer.start();
    }
    for (int i = 0; i < 3; i++) {
        Producer producer = new Producer();
        producer.start();
    }
}
//produce..produce..consume..consume..produce..consume..produce..consume..produce..consume..
```

##### 遇到的问题

```java
Produce..1
Produce..2
consume..3
consume..4
Produce..5
consume..6
consume..7
Produce..8
Produce..9
consume..10
```

输出结果表示消费者在队列为空时产生了消费行为。猜测是消费者在方法内更快的输出结果。

##### 源码分析

###### **ArrayBlockingQueue**

除了不需要集合作参数的构造函数，其他读写方法都使用ReentrantLock锁加锁。ArrayBlockingQueue所有操作公用一个ReentrantLock锁，使用两个condition对象判断队列是否为满队列或空队列。

**enqueue（）方法：**插入元素在当前位置，前进，和信号。只在持有锁时调用。

```java
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;	//为什么要将putIndex置0？答：因为dequeue从队列头部获取元素，所以当队列满后需要从队列头部重新插入
        count++;
        notEmpty.signal();
    }
```

**dequeue（）方法：**提取元素在当前采取位置，前进，和信号。只在持有锁时调用。

```java
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
```

**put（）方法：**将元素插入队列队尾，如果队列已满，则阻塞直到队列有空闲位置。

```java
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```

**add（）方法：**如果可以在不超过队列容量的情况下立即在队列尾部插入指定的元素，成功时返回true，如果队列已满，则抛出IllegalStateException异常。

```java
    public boolean add(E e) {
        return super.add(e);
    }
	//父类AbstractQueue的add方法
    public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }
```

**offer（）方法：**如果可以在不超过队列容量的情况下立即在队列尾部插入指定的元素，成功时返回true，如果队列已满则返回false。这个方法通常比add方法更好，后者只有通过抛出异常才能插入元素。

```java
    public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length)
                return false;
            else {
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
```

**take（）方法：**检查并且出队头元素，如果队列为空则阻塞直到有元素入队。

```java
  public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

**poll（）方法：**检查并出队头元素，如果队列为空则返回null。

```java
public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
```

**poll（long，TimeUnit）方法：**同poll（）方法，多了时间参数，用于设置等待时间。

```java
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);//该方法返回值：nanosTimeout值的估计值减去等待该方法返回所花费的时间。可以使用一个正值作为该方法的后续调用的参数，以完成所需的等待时间。小于或等于0的值表示没有剩余时间。
            }
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

**peek（）方法：**检索但不删除此队列的头，或如果此队列为空，则返回null。

```java
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return itemAt(takeIndex); // null when queue is empty
        } finally {
            lock.unlock();
        }
    }
```

**drainTo（）方法：**从此队列（most）删除给定数量的可用元素，并将它们添加到给定集合中。在尝试向集合c添加元素时遇到的失败可能导致在抛出关联异常时，元素既不在集合中，也不在集合中。试图将队列排空到自身会导致IllegalArgumentException异常。此外，如果在操作过程中修改了指定的集合，则此操作的行为是未定义的。

```java
    public int drainTo(Collection<? super E> c, int maxElements) {
        checkNotNull(c);
        if (c == this)
            throw new IllegalArgumentException();
        if (maxElements <= 0)
            return 0;
        final Object[] items = this.items;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int n = Math.min(maxElements, count);
            int take = takeIndex;
            int i = 0;
            try {
                while (i < n) {
                    @SuppressWarnings("unchecked")
                    E x = (E) items[take];
                    c.add(x);
                    items[take] = null;
                    if (++take == items.length)
                        take = 0;
                    i++;
                }
                return n;
            } finally {
                // Restore invariants even if c.add() threw
                if (i > 0) {
                    count -= i;
                    takeIndex = take;
                    if (itrs != null) {
                        if (count == 0)
                            itrs.queueIsEmpty();
                        else if (i > take)
                            itrs.takeIndexWrapped();
                    }
                    for (; i > 0 && lock.hasWaiters(notFull); i--)
                        notFull.signal();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```

**ArrayBlockingQueue构造方法：**带集合参数的构造方法，可以直接将集合中的元素放入队列中。

```java
/**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity and default access policy.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity < 1}
     */
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity and the specified access policy.
     *
     * @param capacity the capacity of this queue
     * @param fair if {@code true} then queue accesses for threads blocked
     *        on insertion or removal, are processed in FIFO order;
     *        if {@code false} the access order is unspecified.
     * @throws IllegalArgumentException if {@code capacity < 1}
     */
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }

    /**
     * Creates an {@code ArrayBlockingQueue} with the given (fixed)
     * capacity, the specified access policy and initially containing the
     * elements of the given collection,
     * added in traversal order of the collection's iterator.
     *
     * @param capacity the capacity of this queue
     * @param fair if {@code true} then queue accesses for threads blocked
     *        on insertion or removal, are processed in FIFO order;
     *        if {@code false} the access order is unspecified.
     * @param c the collection of elements to initially contain
     * @throws IllegalArgumentException if {@code capacity} is less than
     *         {@code c.size()}, or less than 1.
     * @throws NullPointerException if the specified collection or any
     *         of its elements are null
     */
    public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        this(capacity, fair);

        final ReentrantLock lock = this.lock;
        lock.lock(); // Lock only for visibility, not mutual exclusion
        try {
            int i = 0;
            try {
                for (E e : c) {
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i;
            putIndex = (i == capacity) ? 0 : i;
        } finally {
            lock.unlock();
        }
    }

```

###### LinkedBlockingQueue

##### 暂置-------------------------

#### ForkJoin

主要用于并行计算中，和 MapReduce 原理类似，都是把大的计算任务拆分成多个小任务并行计算。

```java
public class ForkJoinExample extends RecursiveTask<Integer> {

    private final int threshold = 5;
    private int first;
    private int last;

    public ForkJoinExample(int first, int last) {
        this.first = first;
        this.last = last;
    }

    @Override
    protected Integer compute() {
        int result = 0;
        if (last - first <= threshold) {
            // 任务足够小则直接计算
            for (int i = first; i <= last; i++) {
                result += i;
            }
        } else {
            // 拆分成小任务
            int middle = first + (last - first) / 2;
            ForkJoinExample leftTask = new ForkJoinExample(first, middle);
            ForkJoinExample rightTask = new ForkJoinExample(middle + 1, last);
            leftTask.fork();
            rightTask.fork();
            result = leftTask.join() + rightTask.join();
        }
        return result;
    }
}
public static void main(String[] args) throws ExecutionException, InterruptedException {
    ForkJoinExample example = new ForkJoinExample(1, 10000);
    ForkJoinPool forkJoinPool = new ForkJoinPool();
    Future result = forkJoinPool.submit(example);
    System.out.println(result.get());
}
```

ForkJoin 使用 ForkJoinPool 来启动，它是一个特殊的线程池，线程数量取决于 CPU 核数。

```java
public class ForkJoinPool extends AbstractExecutorService
```

ForkJoinPool 实现了工作窃取算法来提高 CPU  的利用率。每个线程都维护了一个双端队列，用来存储需要执行的任务。工作窃取算法允许空闲的线程从其它线程的双端队列中窃取一个任务来执行。窃取的任务必须是最晚的任务，避免和队列所属线程发生竞争。例如下图中，Thread2 从 Thread1 的队列中拿出最晚的 Task1 任务，Thread1 会拿出 Task2  来执行，这样就避免发生竞争。但是如果队列中只有一个任务时还是会发生竞争。

 [![img](https://camo.githubusercontent.com/b06eff6a60482e87b9ded90d6df90f9b47370f84/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f65343266313838662d663461392d346536662d383866632d3435663436383230373266622e706e67)](https://camo.githubusercontent.com/b06eff6a60482e87b9ded90d6df90f9b47370f84/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f65343266313838662d663461392d346536662d383866632d3435663436383230373266622e706e67) 

## 八、线程安全问题

### 不安全

如果多个线程对同一个共享数据进行访问而不采取同步操作的话，那么操作的结果是不一致的。

以下代码演示了 1000 个线程同时对 cnt 执行自增操作，操作结束之后它的值有可能小于 1000。

```java
public class ThreadUnsafeExample {

    private int cnt = 0;

    public void add() {
        cnt++;
    }

    public int get() {
        return cnt;
    }
}
public static void main(String[] args) throws InterruptedException {
    final int threadSize = 1000;
    ThreadUnsafeExample example = new ThreadUnsafeExample();
    final CountDownLatch countDownLatch = new CountDownLatch(threadSize);
    ExecutorService executorService = Executors.newCachedThreadPool();
    for (int i = 0; i < threadSize; i++) {
        executorService.execute(() -> {
            example.add();
            countDownLatch.countDown();
        });
    }
    countDownLatch.await();
    executorService.shutdown();
    System.out.println(example.get());
}
//997
```

### 安全

多个线程不管以何种方式访问某个类，并且在主调代码中不需要进行同步，都能表现正确的行为。

线程安全有以下几种实现方式：

#### 不可变

不可变（Immutable）的对象一定是线程安全的，不需要再采取任何的线程安全保障措施。只要一个不可变的对象被正确地构建出来，永远也不会看到它在多个线程之中处于不一致的状态。多线程环境下，应当尽量使对象成为不可变，来满足线程安全。

不可变的类型：

- final 关键字修饰的基本数据类型
- String
- 枚举类型
- Number 部分子类，如 Long 和 Double 等数值包装类型，BigInteger 和 BigDecimal 等大数据类型。但同为 Number 的原子类 AtomicInteger 和 AtomicLong 则是可变的。

对于集合类型，可以使用 Collections.unmodifiableXXX() 方法来获取一个不可变的集合。UnmodifiableMap<K,V>类是Collections类的内部静态私有类，实现Map和Serializable。

```java
public class ImmutableExample {
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>();
        Map<String, Integer> unmodifiableMap = Collections.unmodifiableMap(map);
        unmodifiableMap.put("a", 1);
    }
}
//Exception in thread "main" java.lang.UnsupportedOperationException
//    at java.util.Collections$UnmodifiableMap.put(Collections.java:1457)
//    at ImmutableExample.main(ImmutableExample.java:9)
```

Collections.unmodifiableXXX() 先对原始的集合进行拷贝，需要对集合进行修改的方法都直接抛出异常。

```java
public V put(K key, V value) {
    throw new UnsupportedOperationException();
}
```

#### 互斥同步

synchronized 和 ReentrantLock。见第四章。

#### 非阻塞同步

互斥同步最主要的问题就是线程阻塞和唤醒所带来的性能问题，因此这种同步也称为阻塞同步。

互斥同步属于一种悲观的并发策略，总是认为只要不去做正确的同步措施，那就肯定会出现问题。无论共享数据是否真的会出现竞争，它都要进行加锁（这里讨论的是概念模型，实际上虚拟机会优化掉很大一部分不必要的加锁）、用户态核心态转换、维护锁计数器和检查是否有被阻塞的线程需要唤醒等操作。

随着硬件指令集的发展，我们可以使用基于冲突检测的乐观并发策略：**先进行操作，如果没有其它线程争用共享数据，那操作就成功了，否则采取补偿措施（不断地重试，直到成功为止）。**这种乐观的并发策略的许多实现都不需要将线程阻塞，因此这种同步操作称为非阻塞同步。

##### 1. CAS

乐观锁需要操作和冲突检测这两个步骤具备原子性，这里就不能再使用互斥同步来保证了，只能靠硬件来完成。硬件支持的原子性操作最典型的是：比较并交换（Compare-and-Swap，CAS）。CAS 指令需要有 3 个操作数，分别是内存地址 V、旧的预期值 A 和新值 B。当执行操作时，只有当 V 的值等于 A，才将 V 的值更新为 B。

CAS的缺点是它使用调用者来处理竞争问题，通过重试、回退、放弃，而锁能自动处理竞争问题，例如阻塞。

CAS（Compare and set）乐观的技术。Java实现的一个compare and set如下，这是一个模拟底层的示例：

```java
@ThreadSafe
public class SimulatedCAS {
    @GuardedBy("this") private int value;

    public synchronized int get() {
        return value;
    }

    public synchronized int compareAndSwap(int expectedValue,
                                           int newValue) {
        int oldValue = value;
        if (oldValue == expectedValue)
            value = newValue;
        return oldValue;
    }

    public synchronized boolean compareAndSet(int expectedValue,
                                              int newValue) {
        return (expectedValue
                == compareAndSwap(expectedValue, newValue));
    }
}
```

非阻塞的计数器：

```java
public class CasCounter {
    private SimulatedCAS value;

    public int getValue() {
        return value.get();
    }

    public int increment() {
        int v;
        do {
            v = value.get();
        } while (v != value.compareAndSwap(v, v + 1));
        return v + 1;
    }
}
```

##### 2. AtomicInteger

J.U.C 包里面的整数原子类 AtomicInteger 的方法调用了 Unsafe 类的 CAS 操作。

以下代码使用了 AtomicInteger 执行了自增的操作。

```java
private AtomicInteger cnt = new AtomicInteger();

public void add() {
    cnt.incrementAndGet();
}
```

以下代码是 incrementAndGet() 的源码，它调用了 Unsafe 的 getAndAddInt() 。

```java
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```

以下代码是 getAndAddInt() 源码，var1 指示对象内存地址，var2 指示该字段相对对象内存地址的偏移，var4  指示操作需要加的数值，这里为 1。通过 getIntVolatile(var1, var2) 得到旧的预期值，通过调用  compareAndSwapInt() 来进行 CAS 比较，如果该字段内存地址中的值等于 var5，那么就更新内存地址为 var1+var2  的变量为 var5+var4。

可以看到 getAndAddInt() 在一个循环中进行，发生冲突的做法是不断的进行重试。

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;//在内存地址中的旧值
}
```

Unsafe是经过特殊处理的，不能理解成常规的java代码，区别在于：

- 1.8在调用getAndAddInt的时候，如果系统底层支持fetch-and-add，那么它执行的就是native方法，使用的是fetch-and-add；
- 如果不支持，就按照上面的所看到的getAndAddInt方法体那样，以java代码的方式去执行，使用的是compare-and-swap；

这也正好跟openjdk8中Unsafe::getAndAddInt上方的注释相吻合：

```javascript
// The following contain CAS-based Java implementations used on
// platforms not supporting native instructions
```

**AtomicInteger如何保证线程安全？**

在AtomicInteger中的incrementAndGet（）方法中，调用了Unsafe类中的getAndAddInt（）方法，该方法中的CAS操作保证了AtomicInteger的线程安全。即保证线程安全是在硬件层面上的CAS操作。

##### 3. ABA

如果一个变量初次读取的时候是 A 值，它的值被改成了 B，后来又被改回为 A，那 CAS 操作就会误认为它从来没有被改变过。

J.U.C 包提供了一个带有标记的原子引用类 AtomicStampedReference  来解决这个问题，它可以通过控制变量值的版本来保证 CAS 的正确性。**大部分情况下 ABA 问题不会影响程序并发的正确性，如果需要解决 ABA  问题，改用传统的互斥同步可能会比原子类更高效。**

##### 知识扩展

**锁的劣势：**

独占，可见性是锁要保证的。

许多JVM都对非竞争的锁获取和释放做了很多优化，性能很不错了。但是如果一些线程被挂起然后稍后恢复运行，当线程恢复后还得等待其他线程执行完他们的时间片，才能被调度，所以挂起和恢复线程存在很大的开销，其实很多锁的力度很小的，很简单，如果锁上存在着激烈的竞争，那么多调度开销/工作开销比值就会非常高。

与锁相比，volatile是一种更轻量级的同步机制，因为使用volatile不会发生上下文切换或者线程调度操作，但是volatile的指明问题就是虽然保证了可见性，但是原子性无法保证，比如i++的字节码就是N行。

锁的其他缺点还包括，如果一个线程正在等待锁，它不能做任何事情，如果一个线程在持有锁的情况下被延迟执行了，例如发生了缺页错误，调度延迟，那么就没法执行。如果被阻塞的线程优先级较高，那么就会出现priority invesion的问题，被永久的阻塞下去。

#### 无同步方案

要保证线程安全，并不是一定就要进行同步。如果一个方法本来就不涉及共享数据，那它自然就无须任何同步措施去保证正确性。

##### 1. 栈封闭

多个线程访问同一个方法的局部变量时，不会出现线程安全问题，因为局部变量存储在虚拟机栈中，属于线程私有的。

```java
public class StackClosedExample {
    public void add100() {
        int cnt = 0;
        for (int i = 0; i < 100; i++) {
            cnt++;
        }
        System.out.println(cnt);
    }
    public static void main(String[] args) {
    	StackClosedExample example = new StackClosedExample();
    	ExecutorService executorService = Executors.newCachedThreadPool();
    	executorService.execute(() -> example.add100());
    	executorService.execute(() -> example.add100());
    	executorService.shutdown();
	}
}
//100
//100
```

##### 2. 线程本地存储（Thread Local Storage）

如果一段代码中所需要的数据必须与其他代码共享，那就看看这些共享数据的代码是否能保证在同一个线程中执行。如果能保证，我们就可以把共享数据的可见范围限制在同一个线程之内，这样，无须同步也能保证线程之间不出现数据争用的问题。

符合这种特点的应用并不少见，大部分使用消费队列的架构模式（如“生产者-消费者”模式）都会将产品的消费过程尽量在一个线程中消费完。其中最重要的一个应用实例就是经典 Web 交互模型中的“一个请求对应一个服务器线程”（Thread-per-Request）的处理方式，这种处理方式的广泛应用使得很多 Web  服务端应用都可以使用线程本地存储来解决线程安全问题。

可以使用 java.lang.ThreadLocal 类来实现线程本地存储功能。

对于以下代码，thread1 中设置 threadLocal 为 1，而 thread2 设置 threadLocal 为 2。过了一段时间之后，thread1 读取 threadLocal 依然是 1，不受 thread2 的影响。

```java
public class ThreadLocalExample {
    public static void main(String[] args) {
        ThreadLocal threadLocal = new ThreadLocal();
        Thread thread1 = new Thread(() -> {
            threadLocal.set(1);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(threadLocal.get());
            threadLocal.remove();
        });
        Thread thread2 = new Thread(() -> {
            threadLocal.set(2);
            threadLocal.remove();
        });
        thread1.start();
        thread2.start();
    }
}
//1
```

为了理解 ThreadLocal，先看以下代码：

```java
public class ThreadLocalExample1 {
    public static void main(String[] args) {
        ThreadLocal threadLocal1 = new ThreadLocal();
        ThreadLocal threadLocal2 = new ThreadLocal();
        Thread thread1 = new Thread(() -> {
            threadLocal1.set(1);
            threadLocal2.set(1);
        });
        Thread thread2 = new Thread(() -> {
            threadLocal1.set(2);
            threadLocal2.set(2);
        });
        thread1.start();
        thread2.start();
    }
}
```

它所对应的底层结构图为：

 [![img](https://camo.githubusercontent.com/8968de525ae8f2712bd1c0c6a5f43a404dc2955c/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f36373832363734632d316266652d343837392d616633392d6539643732326139356433392e706e67)](https://camo.githubusercontent.com/8968de525ae8f2712bd1c0c6a5f43a404dc2955c/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f36373832363734632d316266652d343837392d616633392d6539643732326139356433392e706e67) 



每个 Thread 都有一个 ThreadLocal.ThreadLocalMap 对象。

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

当调用一个 ThreadLocal 的 set(T value) 方法时，先得到当前线程的 ThreadLocalMap 对象，然后将 **ThreadLocal->value 键值对**插入到该 Map 中。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

get() 方法类似。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

ThreadLocal 从理论上讲并不是用来解决多线程并发问题的，因为根本不存在多线程竞争。

在一些场景 (尤其是使用线程池) 下，由于 ThreadLocal.ThreadLocalMap 的底层数据结构导致  ThreadLocal 有内存泄漏的情况，应该尽可能在每次使用 ThreadLocal 后手动调用 remove()，以避免出现  ThreadLocal 经典的内存泄漏甚至是造成自身业务混乱的风险。

##### 3. 可重入代码（Reentrant Code）

这种代码也叫做纯代码（Pure Code），可以在代码执行的任何时刻中断它，转而去执行另外一段代码（包括递归调用它本身），而在控制权返回后，原来的程序不会出现任何错误。

可重入代码有一些共同的特征，例如不依赖存储在堆上的数据和公用的系统资源、用到的状态量都由参数中传入、不调用非可重入的方法等。

## 九、Java内存模型

Java 内存模型试图屏蔽各种硬件和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的内存访问效果。

java内存模型(Java Memory Model，JMM)是 java虚拟机规范定义的，用来屏蔽掉java程序在各种不同的硬件和操作系统对内存的访问的差异，这样就可以实现java程序在各种不同的平台上都能达到内存访问的一致性。可以避免像c++等直接使用物理硬件和操作系统的内存模型在不同操作系统和硬件平台下表现不同，比如有些c/c++程序可能在windows平台运行正常，而在linux平台却运行有问题。

### 主内存与工作内存

处理器上寄存器的读写速度比内存快几个数量级，为了解决这种速度矛盾，在它们之间加入了高速缓存。

加入高速缓存带来了一个新的问题：缓存一致性。如果多个缓存共享同一块主内存区域，那么多个缓存的数据可能会不一致，需要一些协议来解决这个问题。

 [![img](https://camo.githubusercontent.com/7e289aaee2d8533d8870f92659a8984be81eb432/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f39343263613064322d396435632d343561342d383963622d3566643839623631393133662e706e67)](https://camo.githubusercontent.com/7e289aaee2d8533d8870f92659a8984be81eb432/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f39343263613064322d396435632d343561342d383963622d3566643839623631393133662e706e67) 



所有的变量都存储在主内存中，每个线程还有自己的工作内存，**工作内存存储在高速缓存或者寄存器**中，**保存了该线程使用的变量的主内存副本拷贝**。

**线程只能直接操作工作内存中的变量，不同线程之间的变量值传递需要通过主内存来完成。**

 [![img](https://camo.githubusercontent.com/fce232dcfc7411192b429ae86765aa9bef029427/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f31353835313535352d356162632d343937642d616433342d6566656431306634336136622e706e67)](https://camo.githubusercontent.com/fce232dcfc7411192b429ae86765aa9bef029427/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f31353835313535352d356162632d343937642d616433342d6566656431306634336136622e706e67) 

虽然java程序所有的运行都是在虚拟机中，涉及到的内存等信息都是虚拟机的一部分，但实际也是物理机的，只不过是虚拟机作为最外层的容器统一做了处理。虚拟机的内存模型，以及多线程的场景下与物理机的情况是很相似的，可以类比参考。
 **Java内存模型的主要目标是定义程序中变量的访问规则。即在虚拟机中将变量存储到主内存或者将变量从主内存取出这样的底层细节。需要注意的是这里的变量跟我们写 java 程序中的变量不是完全等同的。**这里的变量是指实例字段，静态字段，构成数组对象的元素，但是不包括局部变量和方法参数(因为这是线程私有的)。这里可以简单的认为主内存是java虚拟机内存区域中的堆，局部变量和方法参数是在虚拟机栈中定义的。但是在堆中的变量如果在多线程中都使用，就涉及到了堆和不同虚拟机栈中变量的值的一致性问题了。

Java内存模型中涉及到的概念有：

- 主内存：**java虚拟机规定所有的变量(不是程序中的变量)都必须在主内存中产生**，为了方便理解，可以认为是堆区。可以与前面说的物理机的主内存相比，只不过物理机的主内存是整个机器的内存，而虚拟机的主内存是虚拟机内存中的一部分。
- 工作内存：java虚拟机中每个线程都有自己的工作内存，该内存是线程私有的为了方便理解，可以认为是虚拟机栈。可以与前面说的高速缓存相比。**线程的工作内存保存了线程需要的变量在主内存中的副本。虚拟机规定，线程对主内存变量的修改必须在线程的工作内存中进行，不能直接读写主内存中的变量。**不同的线程之间也不能相互访问对方的工作内存。如果线程之间需要传递变量的值，必须通过主内存来作为中介进行传递。
   **这里需要说明一下：主内存、工作内存与java内存区域中的java堆、虚拟机栈、方法区并不是一个层次的内存划分。这两者是基本上是没有关系的，上文只是为了便于理解，做的类比**

### 内存间交互操作

Java 内存模型定义了 8 个操作来完成主内存和工作内存的交互操作。java虚拟机中主内存和工作内存交互，就是一个变量如何从主内存传输到工作内存中，如何把修改后的变量从工作内存同步回主内存。

 [![img](https://camo.githubusercontent.com/42696b8f0b2cfbf78024f9cd0d806e0e6decb5fe/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38623765626261642d393630342d343337352d383465332d6634313230393964313730632e706e67)](https://camo.githubusercontent.com/42696b8f0b2cfbf78024f9cd0d806e0e6decb5fe/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38623765626261642d393630342d343337352d383465332d6634313230393964313730632e706e67) 



- `read`：作用于主内存变量，表示把一个主内存变量的值传输到线程的工作内存，以便随后的load操作使用。
- `load`：作用于线程的工作内存的变量，表示把read操作从主内存中读取的变量的值放到工作内存的变量副本中(副本是相对于主内存的变量而言的)。
- `use`：作用于线程的工作内存中的变量，表示把工作内存中的一个变量的值传递给执行引擎，**每当虚拟机遇到一个需要使用变量的值的字节码指令时就会执行该操作。**
- `assign`：作用于线程的工作内存的变量，表示把执行引擎返回的结果赋值给工作内存中的变量，**每当虚拟机遇到一个给变量赋值的字节码指令时就会执行该操作。**
- `store`：作用于线程的工作内存中的变量，把工作内存中的一个变量的值传递给主内存，以便随后的write操作使用。
- `write`：作用于主内存的变量，把store操作从工作内存中得到的变量的值放入主内存的变量中。
- `lock`：作用于主内存的变量，一个变量在同一时间只能一个线程锁定，该操作表示这条线成独占这个变量。
- `unlock`：作用于主内存的变量，表示这个变量的状态由处于锁定状态被释放，这样其他线程才能对该变量进行锁定。

如果要把一个变量从主内存传输到工作内存，那就要顺序的执行read和load操作，如果要把一个变量从工作内存回写到主内存，就要顺序的执行store和write操作。**对于普通变量，虚拟机只是要求顺序的执行，并没有要求连续的执行**。对于两个线程，分别从主内存中读取变量a和b的值，并不一样要read a->load a->read b->load b; 也会出现如下执行顺序：read a->read b->load b->load a; (对于volatile修饰的变量会有一些其他规则,后边会详细列出)，对于这8种操作，虚拟机也规定了一系列规则，在执行这8种操作的时候必须遵循如下的规则：

- **不允许`read`和`load`、`store`和`write`操作之一单独出现**，也就是不允许从主内存读取了变量的值但是工作内存不接收的情况，或者不允许从工作内存将变量的值回写到主内存但是主内存不接收的情况。
- **不允许一个线程丢弃最近的`assign`操作**，也就是不允许线程在自己的工作线程中修改了变量的值却不同步/回写到主内存（即线程`use`数据且**修改**后，必须有与之对应的`assign`操作）。
- **不允许一个线程回写没有修改的变量到主内存**，也就是如果线程工作内存中变量没有发生过任何`assign`操作，是不允许将该变量的值回写到主内存（与上一条规则相反）。
- **变量只能在主内存中产生**，不允许在工作内存中直接使用一个未被初始化的变量，也就是没有执行`load`（从主内存获取值）或者`assign`（从执行引擎获取值）操作。也就是说在执行`use`、`store`之前必须对相同的变量执行了`load`、`assign`操作。
- **一个变量在同一时刻只能被一个线程对其进行lock操作**，也就是说一个线程一旦对一个变量加锁后，在该线程没有释放掉锁之前，其他线程是不能对其加锁的，**但是同一个线程对一个变量加锁后，可以继续加锁，同时在释放锁的时候释，放锁次数必须和加锁次数相同。**
- **对变量执行`lock`操作，就会清空工作内存该变量的值**，执行引擎使用这个变量之前，需要重新`load`或者`assign`操作初始化变量的值。
- **不允许对没有`lock`的变量执行`unlock`操作**，如果一个变量没有被`lock`操作，那也不能对其执行`unlock`操作，当然一个线程也不能对被其他线程lock的变量执行`unlock`操作。
- **对一个变量执行`unlock`之前，必须先把变量同步回主内存中**，也就是执行`store`和`write`操作。

当然，最重要的还是如开始所说，这8个动作必须是原子的，不可分割的。

### volatile修饰的变量的特殊规则

关键字`volatile`可以说是Java虚拟机中提供的最轻量级的同步机制。java内存模型对`volatile`专门定义了一些特殊的访问规则。这些规则有些晦涩拗口，先列出规则，然后用更加通俗易懂的语言来解释：
 	假定**T**表示一个线程，**V**和W分别表示两个`volatile`修饰的变量，那么在进行`read`、`load`、`use`、`assign`、`store`和`write`操作的时候需要满足如下规则：

- **只有当线程T对变量V执行的前一个动作是`load`，线程T对变量V才能执行`use`动作；同时只有当线程T对变量V执行的后一个动作是`use`的时候线程T对变量V才能执行`load`操作。**所以，线程**T**对变量**V**的`use`动作和线程**T**对变量**V**的`read`、`load`动作相关联，必须是连续一起出现。也就是在线程**T**的工作内存中，每次使用变量**V**之前必须从主内存去重新获取最新的值（`load`和`read`不能单独出现），**用于保证线程T能看得见其他线程对变量V的最新的修改后的值。**
- **只有当线程T对变量V执行的前一个动作是`assign`的时候，线程T对变量V才能执行`store`动作；同时只有当线程T对变量V执行的后一个动作是`store`的时候，线程T对变量V才能执行assign动作。**所以，线程**T**对变量**V**的`assign`操作和线程**T**对变量**V**的`store`、`write`动作相关联，必须一起连续出现。也即是在线程**T**的工作内存中，每次修改变量**V**之后必须立刻同步回主内存（`store`与`write`不能单独出现），**用于保证线程T对变量V的修改能立刻被其他线程看到。**
- **假定动作A是线程T对变量V实施的`use`或`assign`动作，动作F是和动作A相关联的`load`或`store`动作，动作P是和动作F相对应的对变量V的`read`或`write`动作；类似的，假定动作B是线程T对变量W实施的`use`或`assign`动作，动作G是和动作B相关联的`load`或`store`动作，动作Q是和动作G相对应的对变量W的`read`或`write`动作。如果动作A先于B，那么P先于Q。**也就是说在同一个线程内部，被`volatile`修饰的变量不会被指令重排序，**保证代码的执行顺序和程序的顺序相同**。

总结上面三条规则，前面两条可以概括为：**volatile类型的变量保证对所有线程的可见性**。第三条为：**volatile类型的变量禁止指令重排序优化**。

- **valatile类型的变量保证对所有线程的可见性**
   	可见性是指当一个线程修改了这个变量的值，新值（修改后的值）对于其他线程来说是立即可以得知的。正如上面的前两条规则规定，volatile类型的变量每次值被修改了就立即同步回主内存，每次使用时就需要从主内存重新读取值。返回到前面对普通变量的规则中，并没有要求这一点，所以普通变量的值是不会立即对所有线程可见的。
   	误解：volatile变量对所有线程是立即可见的，所以对volatile变量的所有修改(写操作)都立刻能反应到其他线程中。或者换句话说：volatile变量在各个线程中是一致的，所以基于volatile变量的运算在并发下是线程安全的。
   	这个观点的论据是正确的，但是根据论据得出的结论是错误的，并不能得出这样的结论。volatile的规则，保证了read、load、use的顺序和连续行，同理assign、store、write也是顺序和连续的。也就是这几个动作是原子性的，但是对变量的修改，或者对变量的运算，却不能保证是原子性的（如k++，需要先获取k的值，再进行运算，这两个步骤不能保证原子性）。如果对变量的修改是分为多个步骤的，那么多个线程同时从主内存拿到的值是最新的，但是经过多步运算后回写到主内存的值是有可能存在覆盖情况发生的。如下代码的例子：

```java
public class VolatileTest {
  public static volatile int race = 0;
  public static void increase() {
    race++
  }

  private static final int THREADS_COUNT = 20;

  public void static main(String[] args) {
      Thread[] threads = new Thread[THREADS_COUNT);
      for (int = 0; i < THREADS_COUNT; i++) {
          threads[i] = new Thread(new Runnable(){
              @Override
              public void run() {
                  for (int j = 0; j < 10000; j++) {
                     increase();
                  }
              }
          });
          threads[i].start();
      }
      while (Thread.activeCount() > 1) {
         Thread.yield();
      }
      System.out.println(race);
  }
}
//结果可能小于20W
```

代码就是对volatile类型的变量启动了20个线程，每个线程对变量执行1w次加1操作，如果volatile变量并发操作没有问题的话，那么结果应该是输出20w，但是结果运行的时候每次都是小于20w，这就是因为`race++`操作不是原子性的，是分多个步骤完成的。假设两个线程a、b同时取到了主内存的值，是0，这是没有问题的，在进行`++`操作的时候假设线程a执行到一半，线程b执行完了，这时线程b立即同步给了主内存，主内存的值为1，而线程a此时也执行完了，同步给了主内存，此时的值仍然是1，线程b的结果被覆盖掉了。

volatile不能保证线程安全而synchronized可以保证线程安全。volatile只能保证被其修饰变量的内存可见性,但如果对该变量执行的是非原子操作，线程依旧是不安全的

- **volatile变量禁止指令重排序优化**
   普通的变量仅仅会保证在该方法执行的过程中，所有依赖赋值结果的地方都能获取到正确的结果，但不能保证变量赋值的操作顺序和程序代码的顺序一致。因为在一个线程的方法执行过程中无法感知到这一点，这也就是java内存模型中描述的所谓的“线程内部表现为串行的语义”。
   也就是在单线程内部，我们看到的或者感知到的结果和代码顺序是一致的，即使代码的执行顺序和代码顺序不一致，但是在需要赋值的时候结果也是正确的，所以看起来就是串行的。但实际结果有可能代码的执行顺序和代码顺序是不一致的。这在多线程中就会出现问题。
   看下面的伪代码举例：

```java
Map configOptions;
char[] configText;
//volatile类型bianliang
volatile boolean initialized = false;

//假设以下代码在线程A中执行
//模拟读取配置信息，读取完成后认为是初始化完成
configOptions = new HashMap();
configText = readConfigFile(fileName);
processConfigOptions(configText, configOptions);
initialized = true;

//假设以下代码在线程B中执行
//等待initialized为true后，读取配置信息进行操作
while ( !initialized) {
  sleep();
}
doSomethingWithConfig();

```

如果initialiezd是普通变量，没有被volatile修饰，那么线程A执行的代码的修改初始化完成的结果`initialized = true`就有可能先于之前的三行代码执行，而此时线程B发现initialized为true了，就执行`doSomethingWithConfig()`方法，但是里面的配置信息都是null的，就会出现问题了。
 现在initialized是volatile类型变量，保证禁止代码重排序优化，那么就可以保证`initialized = true`执行的时候，前边的三行代码一定执行完成了，那么线程B读取的配置文件信息就是正确的。

跟其他保证并发安全的工具相比，volatile的性能确实会好一些。在某些情况下，volatile的同步机制性能要优于锁(使用synchronized关键字或者java.util.concurrent包中的锁)。但是现在由于虚拟机对锁的不断优化和实行的许多消除动作，很难有一个量化的比较。
 与自己相比，就可以确定一个原则：volatile变量的读操作和普通变量的读操作几乎没有差异，但是写操作会性能差一些，慢一些，因为要在本地代码中插入许多内存屏障指令来禁止指令重排序，保证处理器不发生代码乱序执行行为。

### long和double变量的特殊规则

Java内存模型要求对主内存和工作内存交换的八个动作是原子的，正如章节开头所讲，对long和double有一些特殊规则。八个动作中lock、unlock、read、load、use、assign、store、write对待32位的基本数据类型都是原子操作，对待long和double这两个64位的数据，java虚拟机规范对java内存模型的规定中特别定义了一条相对宽松的规则：**允许虚拟机将没有被volatile修饰的64位数据的读写操作划分为两次32位的操作来进行，也就是允许虚拟机不保证对64位数据的read、load、store和write这4个动作的操作是原子的。**这也就是我们常说的long和double的非原子性协定(Nonautomic Treatment of double and long Variables)。

### 内存模型三大特性

#### 1. 原子性

Java 内存模型保证了 read、load、use、assign、store、write、lock 和 unlock  操作具有原子性，例如对一个 int 类型的变量执行 assign 赋值操作，这个操作就是原子性的。但是 Java 内存模型允许虚拟机将没有被  volatile 修饰的 64 位数据（long，double）的读写操作划分为两次 32 位的操作来进行，即 load、store、read 和 write 操作可以不具备原子性。目前虚拟机基本都对其实现了原子性。如果需要更大范围的控制，lock和unlock也可以满足需求。lock和unlock虽然没有被虚拟机直接开给用户使用，但是提供了字节码层次的指令monitorenter和monitorexit对应这两个操作，对应到java代码就是synchronized关键字，因此在synchronized块之间的代码都具有原子性。

有一个错误认识就是，int 等原子性的类型在多线程环境中不会出现线程安全问题。前面的线程不安全示例代码中，cnt 属于 int 类型变量，1000 个线程对它进行自增操作之后，得到的值为 997 而不是 1000。

为了方便讨论，将内存间的交互操作简化为 3 个：load、assign、store。

下图演示了两个线程同时对 cnt 进行操作，load、assign、store 这一系列操作整体上看不具备原子性，那么在 T1 修改  cnt 并且还没有将修改后的值写入主内存，T2 依然可以读入旧值。可以看出，这两个线程虽然执行了两次自增运算，但是主内存中 cnt 的值最后为 1 而不是 2。因此对 int 类型读写操作满足原子性只是说明 load、assign、store 这些单个操作具备原子性。

 [![img](https://camo.githubusercontent.com/430b45d369c24a65e4a4641f52b14aaa95d41c7c/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f32373937613630392d363864622d346437622d383730312d3431616339613334623134662e6a7067)](https://camo.githubusercontent.com/430b45d369c24a65e4a4641f52b14aaa95d41c7c/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f32373937613630392d363864622d346437622d383730312d3431616339613334623134662e6a7067) 



AtomicInteger 能保证多个线程修改的原子性。

 [![img](https://camo.githubusercontent.com/b77a1a0dc9c2a35d056d0c76b75df18226283040/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f64643536333033372d666361612d346264382d383362362d6233396439336131326337372e6a7067)](https://camo.githubusercontent.com/b77a1a0dc9c2a35d056d0c76b75df18226283040/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f64643536333033372d666361612d346264382d383362362d6233396439336131326337372e6a7067) 



使用 AtomicInteger 重写之前线程不安全的代码之后得到以下线程安全实现：

```java
public class AtomicExample {
    private AtomicInteger cnt = new AtomicInteger();

    public void add() {
        cnt.incrementAndGet();
    }

    public int get() {
        return cnt.get();
    }
}
public static void main(String[] args) throws InterruptedException {
    final int threadSize = 1000;
    AtomicExample example = new AtomicExample(); // 只修改这条语句
    final CountDownLatch countDownLatch = new CountDownLatch(threadSize);
    ExecutorService executorService = Executors.newCachedThreadPool();
    for (int i = 0; i < threadSize; i++) {
        executorService.execute(() -> {
            example.add();
            countDownLatch.countDown();
        });
    }
    countDownLatch.await();
    executorService.shutdown();
    System.out.println(example.get());
}
//1000
```

除了使用原子类之外，也可以使用 synchronized 互斥锁来保证操作的原子性。它对应的内存间交互操作为：lock 和 unlock，在虚拟机实现上对应的字节码指令为 monitorenter 和 monitorexit。

```java
public class AtomicSynchronizedExample {
    private int cnt = 0;

    public synchronized void add() {
        cnt++;
    }

    public synchronized int get() {
        return cnt;
    }
}
public static void main(String[] args) throws InterruptedException {
    final int threadSize = 1000;
    AtomicSynchronizedExample example = new AtomicSynchronizedExample();
    final CountDownLatch countDownLatch = new CountDownLatch(threadSize);
    ExecutorService executorService = Executors.newCachedThreadPool();
    for (int i = 0; i < threadSize; i++) {
        executorService.execute(() -> {
            example.add();
            countDownLatch.countDown();
        });
    }
    countDownLatch.await();
    executorService.shutdown();
    System.out.println(example.get());
}
//1000
```

#### 2. 可见性

可见性指当一个线程修改了共享变量的值，其它线程能够立即得知这个修改。Java 内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值来实现可见性的。

主要有三种实现可见性的方式：

- volatile
- synchronized，对一个变量执行 unlock 操作之前，必须把变量值同步回主内存。
- final，被 final 关键字修饰的字段在构造器中一旦初始化完成，并且没有发生 this 逃逸（其它线程通过 this 引用访问到初始化了一半的对象），那么其它线程就能看见 final 字段的值。

对前面的线程不安全示例中的 cnt 变量使用 volatile 修饰，不能解决线程不安全问题，因为 volatile 并不能保证操作的原子性。

#### 3. 有序性

有序性是指：在本线程内观察，所有操作都是有序的。在一个线程观察另一个线程，所有操作都是无序的，无序是因为发生了指令重排序。在 Java 内存模型中，允许编译器和处理器对指令进行重排序，重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。

volatile 关键字通过添加内存屏障的方式来禁止指令重排，即重排序时不能把后面的指令放到内存屏障之前。

也可以通过 synchronized 来保证有序性，它保证每个时刻只有一个线程执行同步代码，相当于是让线程顺序执行同步代码。

### 先行发生原则

上面提到了可以用 volatile 和 synchronized 来保证有序性。除此之外，JVM 还规定了先行发生原则，让一个操作无需控制就能先于另一个操作完成，是判断数据是否存在竞争、线程是否安全的主要依据。

#### 1. 单一线程原则

> Single Thread rule

在一个线程内，在程序前面的操作先行发生于后面的操作。

 [![img](https://camo.githubusercontent.com/e1c3998ce9151df8d6b226e640270efff94df669/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38373462336666372d376335632d346537612d623861622d6138326133653033386432302e706e67)](https://camo.githubusercontent.com/e1c3998ce9151df8d6b226e640270efff94df669/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38373462336666372d376335632d346537612d623861622d6138326133653033386432302e706e67) 



#### 2. 管程锁定规则

> Monitor Lock Rule

一个 unlock 操作先行发生于后面对同一个锁的 lock 操作。

 [![img](https://camo.githubusercontent.com/c86b5efc7f2ba505380081f0a343e72a862394f0/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38393936613533372d376334612d346563382d613362372d3765663137393865616532362e706e67)](https://camo.githubusercontent.com/c86b5efc7f2ba505380081f0a343e72a862394f0/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f38393936613533372d376334612d346563382d613362372d3765663137393865616532362e706e67) 

#### 3. volatile 变量规则

> Volatile Variable Rule

对一个 volatile 变量的写操作先行发生于**后面**对这个变量的读操作，这里的后面是指时间上的先后顺序。

 [![img](https://camo.githubusercontent.com/4cba332d162065020d289fbb116e50465ceb0e15/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f39343266333363392d386164392d343938372d383336662d3030376465346332316465302e706e67)](https://camo.githubusercontent.com/4cba332d162065020d289fbb116e50465ceb0e15/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f39343266333363392d386164392d343938372d383336662d3030376465346332316465302e706e67) 

#### 4. 线程启动规则

> Thread Start Rule

Thread 对象的 start() 方法调用先行发生于此线程的**每一个动作**。

 [![img](https://camo.githubusercontent.com/6a64fb5055e471c6de4363bee031982bb42200d9/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f36323730633231362d376563302d346462372d393464652d3030303362636533376364322e706e67)](https://camo.githubusercontent.com/6a64fb5055e471c6de4363bee031982bb42200d9/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f36323730633231362d376563302d346462372d393464652d3030303362636533376364322e706e67) 

#### 5. 线程加入规则

> Thread Join Rule

Thread 对象的结束先行发生于 join() 方法返回。

 [![img](https://camo.githubusercontent.com/17e6baed6955e119c1e512d6ceef339002b343e1/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f32333366386438392d333164372d343133662d396330322d3034326631396334366261312e706e67)](https://camo.githubusercontent.com/17e6baed6955e119c1e512d6ceef339002b343e1/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f32333366386438392d333164372d343133662d396330322d3034326631396334366261312e706e67) 

#### 6. 线程中断规则

> Thread Interruption Rule

对线程 interrupt() 方法的调用先行发生于被中断线程的代码检测到中断事件的发生（catch？？？），可以通过 interrupted() 方法检测到是否有中断发生。

#### 7. 对象终结规则

> Finalizer Rule

一个对象的初始化完成（构造函数执行结束）先行发生于它的 finalize() 方法的开始。

#### 8. 传递性

> Transitivity

如果操作 A 先行发生于操作 B，操作 B 先行发生于操作 C，那么操作 A 先行发生于操作 C。

## 十、锁机制及锁优化

这里的锁优化主要是指 JVM 对 synchronized 的优化。

### 自旋锁

互斥同步进入阻塞状态的开销都很大，应该尽量避免。在许多应用中，共享数据的锁定状态只会持续很短的一段时间。自旋锁的思想是让一个线程在请求一个共享数据的锁时执行忙循环（自旋）一段时间，如果在这段时间内能获得锁，就可以避免进入阻塞状态。

自旋锁虽然能避免进入阻塞状态从而减少开销，但是它需要进行忙循环操作占用 CPU 时间，它**只适用于共享数据的锁定状态很短的场景。**

在 JDK 1.6 中引入了自适应的自旋锁。自适应意味着自旋的次数不再固定了，而是由前一次在同一个锁上的自旋次数及锁的拥有者的状态来决定。

非自旋锁在获取不到锁的时候会进入阻塞状态，从而进入内核态，当获取到锁的时候需要从内核态恢复，需要线程上下文切换。 （线程被阻塞后便进入内核（Linux）调度状态，这个会导致系统在用户态与内核态之间来回切换，严重影响锁的性能）

自旋锁的优点：

1. 自旋锁不会使线程状态发生切换，一直处于用户态，即线程一直都是active的。
2. 不会使线程进入阻塞状态，减少了不必要的上下文切换，执行速度快。

使用自旋锁会有以下一个问题：

1. 如果某个线程持有锁的时间过长，就会导致其它等待获取锁的线程进入循环等待，消耗CPU。使用不当会造成CPU使用率极高。

2. 上面Java实现的自旋锁不是公平的，即无法满足等待时间最长的线程优先获取锁。不公平的锁就会存在“线程饥饿”问题。

**java实现自旋锁的简单例子：**

```java
public class SpinLock {

private AtomicReference cas = new AtomicReference();

public void lock() {

	Thread current = Thread.currentThread();

	// 利用CAS比较引用是否为空，不为空则将值设置为当前线程对象，以实现加锁的作用。
	while (!cas.compareAndSet(null, current)) {
	// DO nothing
	}
}

public void unlock() {
	Thread current = Thread.currentThread();
    //录用CAS比较引用是否为当前线程，如是则设置为空，以实现释放锁的作用。
	cas.compareAndSet(current, null);
	}
}
```

如上代码存在着不可重入的问题，即当一个线程第一次获取到该锁之后，在锁释放之前第二次获取该锁时，就不能成功获取到该锁。由于不满足CAS，所以第二次获取该锁时会进入while忙循环，而如果是可重入锁，第二次也是应该能够获取到该锁的。就极端思想，即使第二次能够成功获取到锁，在第一次释放锁的时候，第二次获取的锁也将被释放。

**可重入自旋锁的实现：**

```java
public class ReentrantSpinLock {

	private AtomicReference cas = new AtomicReference();
	private int count;//引入一个计数器，记录当前线程获取该锁的次数。

	public void lock() {
		Thread current = Thread.currentThread();
		if (current == cas.get()) { 
		// 如果当前线程已经获取到了锁，线程数增加一，然后返回
		count++;
		return;
		}
		// 如果没获取到锁，则通过CAS自旋
		while (!cas.compareAndSet(null, current)) {
			// DO nothing
		}
	}
    
	public void unlock() {
		Thread cur = Thread.currentThread();
		if (cur == cas.get()) {
			if (count > 0) {// 如果大于0，表示当前线程多次获取了该锁，释放锁通过count减一来模拟
				count--;
			} else {
		// 如果count==0，可以将锁释放，这样就能保证获取锁的次数与释放锁的次数是一致的了。
				cas.compareAndSet(cur, null);
			}
		}
	}
}
```

#### 其他自旋锁

##### 1.TicketLock

icketLock主要解决的是公平性的问题。

**思路：**每当有线程获取锁的时候，就给该线程分配一个递增的id，我们称之为排队号，同时，锁对应一个服务号，每当有线程释放锁，服务号就会递增，此时如果服务号与某个线程排队号一致，那么该线程就获得锁，由于排队号是递增的，所以就保证了最先请求获取锁的线程可以最先获取到锁，就实现了公平性。

可以想象成银行办理业务排队，排队的每一个顾客都代表一个需要请求锁的线程，而银行服务窗口表示锁，每当有窗口服务完成就把自己的服务号加一，此时在排队的所有顾客中，只有自己的排队号与服务号一致的才可以得到服务。

实现代码：

```java
public class TicketLock {

    /**
     * 服务号，初始化给定1作为第一号窗口
     */
    private AtomicInteger serviceNum = new AtomicInteger(1);

    /** 
     * 排队号 
     */
    private AtomicInteger ticketNum = new AtomicInteger();

    /** 
     * lock:获取锁，如果获取成功，返回当前线程的排队号，获取排队号用于释放锁. 
     * @return 
     */
    public int lock() {
        int currentTicketNum = ticketNum.incrementAndGet();
        while (currentTicketNum != serviceNum.get()) {
            // Do nothing
        }
        return currentTicketNum;
    }

    /**
     * unlock:释放锁，传入当前持有锁的线程的排队号
     * @param ticketnum
     */
    public void unlock(int ticketnum) {
        serviceNum.compareAndSet(ticketnum, ticketnum + 1);
    }
}
```

上面的实现方式是，线程获取锁之后，将它的排队号返回，等该线程释放锁的时候，需要将该排队号传入。但这样是有风险的，因为这个排队号是可以被修改的，一旦排队号被不小心修改了，那么锁将不能被正确释放。一种更好的实现方式如下（使用ThreadLocal存储线程的排队号（ThreadLocal是线程安全的，但可能会造成内存泄露））:

```java
public class TicketLockV2 {

    /** 
     * 服务号，初始化给定1作为第一号窗口
     */
    private AtomicInteger serviceNum = new AtomicInteger(1);

    /** 
     * 排队号 
     */
    private AtomicInteger ticketNum = new AtomicInteger();

    /** 
     * 新增一个ThreadLocal，用于存储每个线程的排队号 
     */
    private ThreadLocal ticketNumHolder = new ThreadLocal();

    public void lock() {
        int currentTicketNum = ticketNum.incrementAndGet();
		// 获取锁的时候，将当前线程的排队号保存起来
        ticketNumHolder.set(currentTicketNum);
        while (currentTicketNum != serviceNum.get()) {
			// Do nothing
        }
    }

    public void unlock() {
		// 释放锁，从ThreadLocal中获取当前线程的排队号
        Integer currentTickNum = ticketNumHolder.get();
        serviceNum.compareAndSet(currentTickNum, currentTickNum + 1);
    }
}
```

**TicketLock存在的问题**

多处理器系统上，每个进程/线程占用的处理器都在读写同一个变量serviceNum ，每次读写操作都必须在多个处理器缓存之间进行缓存同步，这会导致繁重的系统总线和内存的流量，大大降低系统整体的性能。

下面介绍的MCSLock和CLHLock就是解决这个问题的。

##### **2. CLHLock**

CLH锁是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程只在本地变量上自旋，它不断轮询前驱的状态，如果发现前驱释放了锁就结束自旋，获得锁。

实现代码如下：

```java
import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;

/**
* CLH的发明人是：Craig，Landin and Hagersten。
* 代码来源：http://ifeve.com/java_lock_see2/
*/
public class CLHLock {

	/**
	* 定义一个节点，默认的lock状态为true
	*/
	public static class CLHNode {
		private volatile boolean isLocked = true;
	}

	/**
	* 尾部节点,只用一个节点即可
	*/
	private volatile CLHNode tail;

	private static final ThreadLocal LOCAL = new ThreadLocal();

	private static final AtomicReferenceFieldUpdater UPDATER = 			 AtomicReferenceFieldUpdater.newUpdater(CLHLock.class, CLHNode.class,"tail");

	public void lock() {
		// 新建节点并将节点与当前线程保存起来
		CLHNode node = new CLHNode();
		LOCAL.set(node);
		// 将新建的节点设置为尾部节点（虚拟尾部），并返回旧的节点（原子操作），这里旧的节点实际上就是当前		节点的前驱节点
		CLHNode preNode = UPDATER.getAndSet(this, node);
		if (preNode != null) {
		// 前驱节点不为null表示当前锁被其他线程占用，通过不断轮询判断前驱节点的锁标志位，等待前驱节点释放锁
			while (preNode.isLocked) {
			}
            //为什么需要这两个条语句？防止失误？什么失误？
			preNode = null;
			LOCAL.set(node);
		}
        // 如果不存在前驱节点，表示该锁没有被其他线程占用，则当前线程获得锁	
	}

	public void unlock() {
		// 获取当前线程对应的节点
		CLHNode node = LOCAL.get();
		// 如果tail节点等于node，则将tail节点更新为null，同时将node的lock状态职位false，表示当前线程释放了锁。如果不等于，则代表当前锁不属于该线程。
		if (!UPDATER.compareAndSet(this, node, null)) {
            //将锁标志位设置为false，让其他线程结束自旋。
			node.isLocked = false;
		}
		node = null;
	}
}
```

**CLHLock流程图及其设计到的数据结构**

![CLHLock流程及其数据结构](D:\firefox download\未命名文件(16).jpg)



##### **3. MCSLock**

MCSLock则是对本地变量的节点进行循环。

```java
/**
 * 
 * MCS:发明人名字John Mellor-Crummey和Michael Scott
 * 
 * 代码来源：http://ifeve.com/java_lock_see2/
 * 
 */
public class MCSLock {

    /**
     * 
     * 节点，记录当前节点的锁状态以及后驱节点
     * 
     */
    public static class MCSNode {
        volatile MCSNode next;
        volatile boolean isLocked = true;
    }

    private static final ThreadLocal NODE = new ThreadLocal();

	// 队列
    @SuppressWarnings("unused")
    private volatile MCSNode queue;

	// queue更新器
    private static final AtomicReferenceFieldUpdater UPDATER = AtomicReferenceFieldUpdater.newUpdater(MCSLock.class,MCSNode.class,"queue");

    public void lock() {
		// 创建节点并保存到ThreadLocal中
        MCSNode currentNode = new MCSNode();
        NODE.set(currentNode);
		// 将queue设置为当前节点，并且返回之前的节点
        MCSNode preNode = UPDATER.getAndSet(this, currentNode);
        if (preNode != null) {
			// 如果之前节点不为null，表示锁已经被其他线程持有
            preNode.next = currentNode;
			// 循环判断，直到当前节点的锁标志位为false（前驱节点释放锁时会将当前节点锁标志位置false）。
            while (currentNode.isLocked) {
            }
        }
    }

    public void unlock() {
        MCSNode currentNode = NODE.get();
		// next为null表示没有正在等待获取锁的线程
        if (currentNode.next == null) {
		// 更新状态并设置queue为null
            if (UPDATER.compareAndSet(this, currentNode, null)) {
				// 如果成功了，表示queue==currentNode,即当前节点后面没有节点了
                return;
            } else {
				// 如果不成功，表示queue!=currentNode,即当前节点后面多了一个节点，表示有线程在等待
				// 如果当前节点的后续节点为null，则需要等待其不为null（参考加锁方法）
                // currentNode.next==null的情况是：其他线程在获取锁时将更新器设置为其当前节点，但为进			入判断后的preNode.next=currentNode的语句。
                while (currentNode.next == null) {
                }
            }
        } else {
			// 如果不为null，表示有线程在等待获取锁，此时将等待线程对应的节点锁状态更新为false，同时将当	前线程的后继节点设为null
            currentNode.next.isLocked = false;
            currentNode.next = null;
        }
    }
}
```

##### **4. CLHLock 和 MCSLock**

都是基于链表，不同的是CLHLock是基于隐式链表，没有真正的后续节点属性，MCSLock是显式链表（伪？），有一个指向后续节点的属性。

将获取锁的线程状态借助节点(node)保存,每个线程都有一份独立的节点，这样就解决了TicketLock多处理器缓存同步的问题。（自适应的自旋锁）

**自旋锁与互斥锁**

自旋锁与互斥锁都是为了实现保护资源共享的机制。

无论是自旋锁还是互斥锁，在任意时刻，都最多只能有一个保持者。

获取互斥锁的线程，如果锁已经被占用，则该线程将进入睡眠状态；获取自旋锁的线程则不会睡眠，而是一直循环等待锁释放。

##### 总结

**自旋锁**：线程获取锁的时候，如果锁被其他线程持有，则当前线程将循环等待，直到获取到锁。

自旋锁等待期间，线程的状态不会改变，线程一直是用户态并且是活动的(active)。

自旋锁如果持有锁的时间太长，则会导致其它等待获取锁的线程耗尽CPU。

自旋锁本身无法保证公平性，同时也无法保证可重入性。

基于自旋锁，可以实现具备公平性和可重入性质的锁。

TicketLock:采用类似银行排号叫好的方式实现自旋锁的公平性，但是由于不停的读取serviceNum，每次读写操作都必须在多个处理器缓存之间进行缓存同步，这会导致繁重的系统总线和内存的流量，大大降低系统整体的性能。

CLHLock和MCSLock通过链表的方式避免了减少了处理器缓存同步，极大的提高了性能，区别在于CLHLock是通过轮询其前驱节点的状态，而MCS则是查看当前节点的锁状态。

CLHLock在NUMA架构下使用会存在问题。在没有cache的NUMA系统架构中，由于CLHLock是在当前节点的前一个节点上自旋,**NUMA架构中处理器访问本地内存的速度高于通过网络访问其他节点的内存**，所以CLHLock在NUMA架构上不是最优的自旋锁。

##### NUMA架构简单介绍

NUMA*（Non Uniform Memory Access Architecture）*技术可以使众多服务器像单一系统那样运转，同时保留小系统便于编程和管理的优点。基于电子商务应用对内存访问提出的更高的要求，NUMA也向复杂的结构设计提出了挑战。

### 锁消除

锁消除是Java虚拟机在JIT编译（即时编译）时，通过对运行上下文的扫描，去除不可能存在共享资源竞争的锁，通过锁消除，可以节省毫无意义的请求锁时间。

锁消除主要是通过逃逸分析来支持，如果堆上的共享数据不可能逃逸出去被其它线程访问到，那么就可以把它们当成私有数据对待，也就可以将它们的锁进行消除。

锁消除是发生在编译器级别的一种锁优化方式。
有时候我们写的代码完全不需要加锁，却执行了加锁操作。

我们可以通过编译器将其优化，将锁消除，前提是java必须运行在server模式（server模式会比client模式作更多的优化），同时必须开启逃逸分析:

-server -XX:+DoEscapeAnalysis -XX:+EliminateLocks

对于一些看起来没有加锁的代码，其实隐式的加了很多锁。例如下面的字符串拼接代码就隐式加了锁：

```java
public static String concatString(String s1, String s2, String s3) {
    return s1 + s2 + s3;
}
```

String 是一个不可变的类，编译器会对 String 的拼接自动优化。在 JDK 1.5 之前，会转化为 StringBuffer 对象的连续 append() 操作：

```java
public static String concatString(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}
```

每个 append() 方法中都有一个同步块。虚拟机观察变量 sb，很快就会发现它的动态作用域被限制在 concatString()  方法内部。也就是说，sb 的所有引用永远不会逃逸到 concatString() 方法之外，其他线程无法访问到它，因此可以进行消除。

#### 逃逸分析介绍

逃逸分析的基本行为就是分析对象动态作用域：当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他地方中，称为方法逃逸。逃逸分析 从jdk 1.7开始已经默认开始逃逸分析。

例如以下代码：

```java
public static StringBuffer craeteStringBuffer(String s1, String s2) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    return sb;
}

public static String createStringBuffer(String s1, String s2) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    return sb.toString();
}

```

第一段代码中的`sb`就逃逸了，而第二段代码中的`sb`就没有逃逸。因为第一段代码的sb对象作返回值返回给其他方法调用。

一、同步省略。如果一个对象被发现只能从一个线程被访问到，那么对于这个对象的操作可以不考虑同步。

二、将堆分配转化为栈分配。如果一个对象在子程序中被分配，要使指向该对象的指针永远不会逃逸，对象可能是栈分配的候选，而不是堆分配。

三、分离对象或标量替换。有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而是存储在CPU寄存器中。

##### 同步省略

在动态编译同步块时，JIT编译器可以借助逃逸分析来判断同步块所使用的锁对象是否只在一个线程中被访问而没有逃逸到其他线程中。

如果同步块所使用的锁对象只能被一个线程访问，那么JIT编译器在编译这个同步块的时候就回取消对这部分代码的同步。这个取消同步的过程叫同步省略，也叫锁消除。

如下代码：

```java
    public static void synLock() {
        Integer syn = new Integer(0);
        synchronized(syn) {
            syn++;
        }
    }
```

代码中对syn对象进行加锁，但是syn对象生命周期只在synLock（）方法中，并不会被其他线程访问到，所以在JIT编译阶段就会被优化成如下代码：

```java
    public static void synLock() {
        Integer syn = new Integer(0);
        syn++;
    }
```

所以，在使用synchronized的时候，如果JIT经过逃逸分析之后发现并无线程安全问题的话，就会做锁消除。

##### 标量替换

标量（Scalar）是指一个无法再分解成更小的数据的数据。Java中的原始数据类型就是标量。相对的，那些还可以分解的数据叫做聚合量（Aggregate），Java中的对象就是聚合量，因此它可以分解成其他聚合量和标量。

在JIT阶段，如果经过逃逸分析，发现一个对象不会被外界访问的话，那么经过JIT优化，就会把这个对象拆解成若干个其中包含的若干个成员变量来代替。这个过程就是标量替换。

```java
public static void main(String[] args) {
   alloc();
}

private static void alloc() {
   Point point = new Point（1,2）;
   System.out.println("point.x="+point.x+"; point.y="+point.y);
}
class Point{
    private int x;
    private int y;
}
```

以上代码中，point对象并没有逃逸出alloc（）方法，并且point对象是可以拆解成标量的。那么，JIT就不会直接创建Point对象，而是直接使用两个标量int x，int y来代替Point对象。

经过标量替换后，代码会变成：

```java
private static void alloc() {
   int x = 1;
   int y = 2;
   System.out.println("point.x="+x+"; point.y="+y);
}
```

可以看到，Point这个聚合量经过逃逸分析后，发现他并没有逃逸，就被替换成两个标量了。那么标量替换有什么好处呢？就是可以大大减少堆内存的占用。因为一旦不需要创建对象了，那么就不再需要分配堆内存了。

标量替换为栈上分配提供了很好的基础。

##### 栈上分配

在Java虚拟机中，对象是在Java堆中分配内存的，这是一个普遍的常识。但是，有一种特殊情况，那就是如果经过逃逸分析后发现，一个对象并没有逃逸出方法的话，那么就可能被优化成栈上分配。这样就无需在堆上分配内存，也无须进行垃圾回收了。

关于栈上分配的详细介绍，可以参考**对象和数组并不是都在堆上分配内存的**。

这里，还是要简单说一下，其实在现有的虚拟机中，并没有真正的实现栈上分配，**在对象和数组并不是都在堆上分配内存的**中我们的例子中，对象没有在堆上分配，其实是标量替换实现的。

##### 逃逸分析弊端

关于逃逸分析的论文在1999年就已经发表了，但直到JDK 1.6才有实现，而且这项技术到如今也并不是十分成熟的。

其根本原因就是无法保证逃逸分析的性能消耗一定能高于他的消耗。虽然经过逃逸分析可以做标量替换、栈上分配、和锁消除。但是逃逸分析自身也是需要进行一系列复杂的分析的，这其实也是一个相对耗时的过程。

一个极端的例子，就是经过逃逸分析之后，发现没有一个对象是不逃逸的。那这个逃逸分析的过程就白白浪费掉了。

### 锁粗化

如果一系列的连续操作都对同一个对象反复加锁和解锁，频繁的加锁操作就会导致性能损耗。

```java
    public String cancatString(String s1, String s2, String s3){
        StringBuffer sb = new StringBuffer();
        sb.append(s1);
        sb.append(s2);
        sb.append(s3);
        return sb.toString();
    }

```

如上的示例代码中连续的 append()  方法就属于这类情况。如果虚拟机探测到由这样的一串零碎的操作都对同一个对象加锁，将会把加锁的范围扩展（粗化）到整个操作序列的外部。对于如上的示例代码就是扩展到第一个 append() 操作之前直至最后一个 append() 操作之后，这样只需要加锁一次就可以了。

### 轻量级锁

1.6 引入了偏向锁和轻量级锁，从而让锁拥有了四个状态：无锁状态（unlocked）、偏向锁状态（biasble）、轻量级锁状态（lightweight locked）和重量级锁状态（inflated）。

以下是 HotSpot 虚拟机对象头的内存布局，这些数据被称为 Mark Word。其中 tag bits 对应了五个状态，这些状态在右侧的 state 表格中给出。除了 marked for gc 状态，其它四个状态已经在前面介绍过了。

 [![img](https://camo.githubusercontent.com/deab9b2c52091554cc095249fd7a97fc1ca41521/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f62623661343962652d303066322d346632372d613063652d3465643736346263363035632e706e67)](https://camo.githubusercontent.com/deab9b2c52091554cc095249fd7a97fc1ca41521/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f62623661343962652d303066322d346632372d613063652d3465643736346263363035632e706e67) 



下图左侧是一个线程的虚拟机栈，其中有一部分称为 Lock Record 的区域，这是在轻量级锁运行过程创建的，用于存放锁对象的 Mark Word。而右侧就是一个锁对象，包含了 Mark Word 和其它信息。

 [![img](https://camo.githubusercontent.com/c1a0307ca4be2bc4b5a3492c634553b4255759bd/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f30353165343336632d306534362d346335392d386636372d3532643839643635363138322e706e67)](https://camo.githubusercontent.com/c1a0307ca4be2bc4b5a3492c634553b4255759bd/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f30353165343336632d306534362d346335392d386636372d3532643839643635363138322e706e67) 



轻量级锁是相对于传统的重量级锁而言，它使用 CAS 操作来避免重量级锁使用互斥量的开销。对于绝大部分的锁，在整个同步周期内都是不存在竞争的，因此也就不需要都使用互斥量进行同步，可以先采用 CAS 操作进行同步，如果 CAS 失败了再改用互斥量进行同步。

当尝试获取一个锁对象时，如果锁对象标记为 0 01，说明锁对象的锁未锁定（unlocked）状态。此时虚拟机在当前线程的虚拟机栈中创建  Lock Record，然后使用 CAS 操作将对象的 Mark Word 更新为 Lock Record 指针。如果 CAS  操作成功了，那么线程就获取了该对象上的锁，并且对象的 Mark Word 的锁标记变为 00，表示该对象处于轻量级锁状态。

 [![img](https://camo.githubusercontent.com/740f6e8b42fda5eaca9b83b919a1729059653705/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f62616161363831662d376335322d343139382d613561652d3330336239333836636634372e706e67)](https://camo.githubusercontent.com/740f6e8b42fda5eaca9b83b919a1729059653705/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f62616161363831662d376335322d343139382d613561652d3330336239333836636634372e706e67) 



如果 CAS 操作失败了，虚拟机首先会检查对象的 Mark Word  是否指向当前线程的虚拟机栈，如果是的话说明当前线程已经拥有了这个锁对象，那就可以直接进入同步块继续执行，否则说明这个锁对象已经被其他线程线程抢占了。如果有两条以上的线程争用同一个锁，那轻量级锁就不再有效，要膨胀为重量级锁。

### 偏向锁

偏向锁的思想是偏向于让第一个获取锁对象的线程，这个线程在之后获取该锁就不再需要进行同步操作，甚至连 CAS 操作也不再需要。

当锁对象第一次被线程获得的时候，进入偏向状态，标记为 1 01。同时使用 CAS 操作将线程 ID 记录到 Mark Word 中，如果 CAS 操作成功，这个线程以后每次进入这个锁相关的同步块就不需要再进行任何同步操作。

当有另外一个线程去尝试获取这个锁对象时，偏向状态就宣告结束，此时撤销偏向（Revoke Bias）后恢复到未锁定状态或者轻量级锁状态。

 [![img](https://camo.githubusercontent.com/8369c6bebffb9617694168381f5d7a62bb099e62/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f33393063393133622d356633312d343434662d626264622d3262383862363838653763652e6a7067)](https://camo.githubusercontent.com/8369c6bebffb9617694168381f5d7a62bb099e62/68747470733a2f2f63732d6e6f7465732d313235363130393739362e636f732e61702d6775616e677a686f752e6d7971636c6f75642e636f6d2f33393063393133622d356633312d343434662d626264622d3262383862363838653763652e6a7067) 

### 锁过程及MarkWord

![img](https://upload-images.jianshu.io/upload_images/4491294-e3bcefb2bacea224.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

线程访问同步代码块时，判断当前锁对象是处于何种状态：

- 01（标志位）：判断是否为偏向锁————1 01（偏向锁）或 0 01（无锁状态）：

  1.偏向锁（1 01）：检查对象头的Mark Word中记录的是否是当前线程ID：

  ​	1.1.是：获得偏向锁（Thread ID | epoch | age | （偏向锁）1 | （标志位）01）——》执行同步代码块；

  ​	1.2.否：CAS操作将锁对象中的Thread ID 替换为当前线程ID：

  ​		1.2.1.成功：获得偏向锁（Thread ID | epoch | age | （偏向锁）1 | （标志位）01）——》执行同步代码块；

  ​		1.2.2.失败：其他线程尝试获取这个对象，开始偏向锁撤销（等待竞争出现才释放锁的机制），当原持有偏向锁的线程达到安全点（没有线程在执行字节码），判断原持有偏向锁的线程处于什么线程状态：

  ​			1.2.2.1.未退出同步代码块：升级为轻量级锁，更新原持有偏向锁及线程的Mark Word，该状态表示原持有偏向锁线程持有该偏向锁，需要将该锁膨胀为轻量级锁，进入轻量级锁的步骤；

  ​			1.2.2.2.未活动状态/已退出同步代码块：原持有偏向锁线程释放锁 （空 | （无锁）0 | （标志位）01），唤醒原持有偏向锁的线程，该状态表示原持有偏向锁线程不持有该偏向锁，会撤销对象回到可偏向但还没有偏向的状态，然后重新尝试获取锁；

  2.无锁状态（0 01）：跳到CAS操作将锁对象中的Thread ID 替换为当前线程ID的步骤...

- 轻量级锁 00（标志位）：操作当前线程的栈中分配记录，拷贝对象头中的Mark Word到当前线程的锁记录中，尝试CAS操作，将对象头的Mark Word中锁记录指针指向当前线程锁记录：

  ​	1.成功：获取轻量级锁（指向当前线程锁记录的指针 | （标志位）00），执行同步代码块，开始尝试轻量级锁解锁。使用CAS操作解锁，满足两个CAS操作：一、对象头中的Mark Word中锁记录指针是否仍是指向当前线程锁记录（判断该锁是否被其他线程争用）；二、拷贝在当前线程锁记录的Mark Word信息是否与头像中的Mark Word一致（判断是否是同一个锁）；

  ​		1.1.成功：释放锁；

  ​		1.2.失败：表示正在有线程争用该锁，会以一种“非常慢”的方式来正确的释放锁并通知其他等待线程来获取锁，开始新一轮竞争；

  ​	2.失败：自旋尝试获取锁，达到一定次数CAS操作后仍然未成功则膨胀为重量级锁（有自适应的自旋锁）

- 重量级锁 10（标志位）：重量级锁（指向重量级锁monitor的指针 | （标志位）10），争用mutex（互斥体），挂起当前线程，等待唤醒；

## 十一、并发优化

![img](https://images2018.cnblogs.com/blog/1285727/201806/1285727-20180625070314183-2036349251.png)

### 锁优化

#### 减少锁的持有时间

例如避免给整个方法加锁

```java
public synchronized void syncMethod(){ 
	othercode1(); 
	mutextMethod(); 
	othercode2(); 
}
```

改进后

```java
public void syncMethod2(){ 
	othercode1(); 
	synchronized(this){ 
		mutextMethod(); 
	} 
	othercode2(); 
}
```

#### 减小锁的粒度

将大对象，拆成小对象，大大增加并行度，降低锁竞争. 如此一来偏向锁，轻量级锁成功率提高. 

一个简单的例子就是jdk内置的ConcurrentHashMap与SynchronizedMap。

##### **Collections.synchronizedMap源码分析**

Collections工具类的静态内部类，对构造函数传入的Collection的所有方法都对mutex进行加锁，作用于同一对象。

**字段：**

```java
private final Map<K,V> m;     // Backing Map
final Object      mutex;        // Object on which to synchronize
private transient Set<K> keySet;
private transient Set<Map.Entry<K,V>> entrySet;
private transient Collection<V> values;
```

其构造函数有：

```java
		SynchronizedCollection(Collection<E> c) {
            this.c = Objects.requireNonNull(c);
            mutex = this;
        }

        SynchronizedCollection(Collection<E> c, Object mutex) {
            this.c = Objects.requireNonNull(c);
            this.mutex = Objects.requireNonNull(mutex);
        }
```

其本质是在读写map操作上都加了锁, 在高并发下性能一般.

```java
        public int size() {
            synchronized (mutex) {return m.size();}
        }
        public boolean isEmpty() {
            synchronized (mutex) {return m.isEmpty();}
        }
        public boolean containsKey(Object key) {
            synchronized (mutex) {return m.containsKey(key);}
        }
        public boolean containsValue(Object value) {
            synchronized (mutex) {return m.containsValue(value);}
        }
        public V get(Object key) {
            synchronized (mutex) {return m.get(key);}
        }

        public V put(K key, V value) {
            synchronized (mutex) {return m.put(key, value);}
        }
        public V remove(Object key) {
            synchronized (mutex) {return m.remove(key);}
        }
        public void putAll(Map<? extends K, ? extends V> map) {
            synchronized (mutex) {m.putAll(map);}
        }
        public void clear() {
            synchronized (mutex) {m.clear();}
        }
```

##### **ConcurrentHashMap源码分析**

**JDK-1.7版本：**

JDK1.7的实现是**数组+Segment+分段锁**的方式。

**1.Segment（分段锁）**

ConcurrentHashMap中的**分段锁称为Segment**，它即类似于HashMap的结构，即内部拥有一个Entry数组，数组中的每个元素又是一个链表,同时又是一个ReentrantLock（Segment继承了ReentrantLock）。

**2.内部结构**

ConcurrentHashMap使用分段锁技术，将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问，能够实现真正的并发访问。如下图是ConcurrentHashMap的内部结构图：

[![彻底搞清楚ConcurrentHashMap的实现原理(含JDK1.7和JDK1.8的区别)](https://youzhixueyuan.com/blog/wp-content/uploads/2018/11/2020050202270792.png)](https://youzhixueyuan.com/blog/wp-content/uploads/2018/11/2020050202270792.png)

面的结构我们可以了解到，ConcurrentHashMap定位一个元素的过程需要进行两次Hash操作。

**第一次Hash定位到Segment，第二次Hash定位到元素所在的链表的头部。**

**3.该结构的优劣势**

**坏处**

这一种结构的带来的副作用是Hash的过程要比普通的HashMap要长

**好处**

写操作的时候可以只对元素所在的Segment进行加锁即可，不会影响到其他的Segment，这样，在最理想的情况下，ConcurrentHashMap可以最高同时支持Segment数量大小的写操作（刚好这些写操作都非常平均地分布在所有的Segment上）。

所以，通过这一种结构，ConcurrentHashMap的并发能力可以大大的提高。



内部使用分区Segment来表示不同的部分, 每个分区其实就是一个小的hashtable. 各自有自己的锁。

只要多个修改发生在不同的分区, 他们就可以并发的进行. 把一个整体分成了16个Segment, 最高支持16个线程并发修改。

代码中运用了很多volatile声明共享变量, 第一时间获取修改的内容, 性能较好。



**JDK-1.8版本：**

JDK1.8采用了**数组+链表+红黑树**的实现方式。

[![彻底搞清楚ConcurrentHashMap的实现原理(含JDK1.7和JDK1.8的区别)](https://youzhixueyuan.com/blog/wp-content/uploads/2018/11/2020050202351287.png)](https://youzhixueyuan.com/blog/wp-content/uploads/2018/11/2020050202351287.png)

**内部大量采用CAS操作，这里我简要介绍下CAS。**

CAS是compare and  swap的缩写，即我们所说的比较交换。cas是一种基于锁的操作，而且是乐观锁。在java中锁分为乐观锁和悲观锁。悲观锁是将资源锁住，等一个之前获得锁的线程释放锁之后，下一个线程才可以访问。而乐观锁采取了一种宽泛的态度，通过某种方式不加锁来处理资源，比如通过给记录加version来获取数据，性能较悲观锁有很大的提高。

CAS 操作包含三个操作数 ——  内存位置（V）、预期原值（A）和新值(B)。如果内存地址里面的值和A的值是一样的，那么就将内存里面的值更新成B。CAS是通过无限循环来获取数据的，若果在第一轮循环中，a线程获取地址里面的值被b线程修改了，那么a线程需要自旋，到下次循环才有可能机会执行。

**JDK8中彻底放弃了Segment转而采用的是Node，其设计思想也不再是JDK1.7中的分段锁思想。**

**Node：保存key，value及key的hash值的数据结构。其中value和next都用volatile修饰，保证并发的可见性。**

```java
    /**
     * Key-value entry.  This class is never exported out as a
     * user-mutable Map.Entry (i.e., one supporting setValue; see
     * MapEntry below), but can be used for read-only traversals used
     * in bulk tasks.  Subclasses of Node with a negative hash field
     * are special, and contain null keys and values (but are never
     * exported).  Otherwise, keys and vals are never null.
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        volatile V val;
        volatile Node<K,V> next;
	//省略部分代码...
    }
```

**Java8 ConcurrentHashMap结构基本上和Java8的HashMap一样，不过保证线程安全性。**

在JDK8中ConcurrentHashMap的结构，由于引入了红黑树，使得ConcurrentHashMap的实现非常复杂，我们都知道，红黑树是一种性能非常好的二叉查找树，其查找性能为O（logN），但是其实现过程也非常复杂，而且可读性也非常差，Doug Lea的思维能力确实不是一般人能比的，早期完全采用链表结构时Map的查找时间复杂度为O（N），JDK8中ConcurrentHashMap在链表的长度大于某个阈值的时候会将链表转换成红黑树进一步提高其查找性能。

![彻底搞清楚ConcurrentHashMap的实现原理(含JDK1.7和JDK1.8的区别)](https://youzhixueyuan.com/blog/wp-content/uploads/2019/07/20190730164533_34126.jpg)



**部分字段：**

所有属性都使用volatile修饰。

```java
	/**
     * 节点数组。在第一次插入时惰性初始化。大小总是2的幂。通过迭代器直接访问。
     */
    transient volatile Node<K,V>[] table;

    /**
     * 使用下一个table; 只有在调整大小时是非空的。
     */
    private transient volatile Node<K,V>[] nextTable;

    /**
     * 基本计数器值，主要在没有争用时使用，但也用作表初始化竞争期间的回退。通过CAS更新数据。
     */
    private transient volatile long baseCount;

    /**
     * 表初始化和调整大小控件。当为负数时，表正在初始化或调整大小:-1表示初始化，否则为-(1 +活跃的正在调整大 	   * 小的线程数)。否则，当表为null时，保存创建时要使用的初始表大小，或默认为0。初始化之后，保存要调整表大	   * 小的下一个元素计数值。
     */
    private transient volatile int sizeCtl;

    /**
     * 调整大小时要分割的下一个表索引(加1)。
     */
    private transient volatile int transferIndex;

    /**
     * 自旋锁(通过CAS锁定)在调整大小和/或创建反单元格时使用。
     */
    private transient volatile int cellsBusy;
    /**
     * 计数器单元表。当非空时，大小是2的乘方。
     */
    private transient volatile CounterCell[] counterCells;
```

**构造函数：**

```java
    /**
     * Creates a new, empty map with an initial table size
     * accommodating the specified number of elements without the need
     * to dynamically resize.
     *
     * @param initialCapacity The implementation performs internal
     * sizing to accommodate this many elements.
     * @throws IllegalArgumentException if the initial capacity of
     * elements is negative
     */
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
    /**
     * Creates a new map with the same mappings as the given map.
     *
     * @param m the map
     */
    public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
        this.sizeCtl = DEFAULT_CAPACITY;
        putAll(m);
    }

    /**
     * Creates a new, empty map with an initial table size based on
     * the given number of elements ({@code initialCapacity}) and
     * initial table density ({@code loadFactor}).
     *
     * @param initialCapacity the initial capacity. The implementation
     * performs internal sizing to accommodate this many elements,
     * given the specified load factor.
     * @param loadFactor the load factor (table density) for
     * establishing the initial table size
     * @throws IllegalArgumentException if the initial capacity of
     * elements is negative or the load factor is nonpositive
     *
     * @since 1.6
     */
    public ConcurrentHashMap(int initialCapacity, float loadFactor) {
        this(initialCapacity, loadFactor, 1);
    }

    /**
     * Creates a new, empty map with an initial table size based on
     * the given number of elements ({@code initialCapacity}), table
     * density ({@code loadFactor}), and number of concurrently
     * updating threads ({@code concurrencyLevel}).
     *
     * @param initialCapacity the initial capacity. The implementation
     * performs internal sizing to accommodate this many elements,
     * given the specified load factor.
     * @param loadFactor the load factor (table density) for
     * establishing the initial table size
     * @param concurrencyLevel the estimated number of concurrently
     * updating threads. The implementation may use this value as
     * a sizing hint.
     * @throws IllegalArgumentException if the initial capacity is
     * negative or the load factor or concurrencyLevel are
     * nonpositive
     */
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);
        this.sizeCtl = cap;
    }
```

###### **ConcurrentHashMap与HashMap初始化比较**

HashMap的resize（）方法，在初始化或者调整数组大小时，并没有考虑线程安全的问题。

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

ConcurrentHashMap在初始化时，会进行table数组的非空判断，如果没有值则进行下面初始化的步骤。在while循环里面围绕着sizeCtl变量调整table数组的大小或初始化。使用CAS操作保证第一个进入的线程完成初始化，其他线程则退出初始化方法。

```java
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

###### **ConcurrentHashMap与HashMap添加元时比较**

HashMap添加元素时，判断节点是否为空，若空则添加元素，非空则在链表后插入元素。该方法如果在多线程同时进入的情况下，就会出现数据存储问题。

```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

ConcurrentHashMap在添加元素时，判断如果table中对应的位置为null，则使用casTabAt方法进行CAS操作将元素节点添加到table中。 而如果对应位置不为null，则使用synchronized代码块的形式同步添加元素节点。

synchronized 里的 f 代表的就是当前节点，所以这个synchronized 只是对当前节点加了锁。这就是ConcurrentHashMao在处理在对同一个数组中的元素操作时对线程安全问题的处理。

```java
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```

###### **ConcurrentHashMap与HashMap在扩容时比较**

HashMap在数组的元素过多时会进行扩容操作，扩容之后会把原数组中的元素拿到新的数组中，这时候在多线程情况下就有可能出现多个线程搬运一个元素。或者说一个线程正在进行扩容，但是另一个线程还想进来存或者读元素，这也可会出现线程安全问题。在ConcurrentHashMap中：

```java
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
           省略......
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            省略......
    }
    addCount(1L, binCount);
    return null;
}
```

我们看到 (fh = f.hash) == MOVED 有这样一个判断，MOVED  是一个成员静态变量，值为-1，当数组在扩容的时候会把数组的头节点的hash值变为-1，所以当线程进来不管是查询还是修改还是添加只要看到当前主节点的hash值为-1时就会进入这里面的方法。

```java
/**
 * 如果当前节点正在重新分配元素，则帮助它
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```



###### 总结

其实可以看出JDK1.8版本的ConcurrentHashMap的数据结构已经接近HashMap，相对而言，ConcurrentHashMap只是增加了同步的操作来控制并发，从JDK1.7版本的ReentrantLock+Segment+HashEntry，到JDK1.8版本中synchronized+CAS+HashEntry+红黑树。

**1.数据结构**：取消了Segment分段锁的数据结构，取而代之的是数组+链表+红黑树的结构。

**2.保证线程安全机制**：JDK1.7采用segment的分段锁机制实现线程安全，其中segment继承自ReentrantLock。JDK1.8采用CAS+Synchronized保证线程安全。

 **3.锁的粒度**：原来是对需要进行数据操作的Segment加锁，现调整为对每个数组元素加锁（Node）。

**4.链表转化为红黑树**:定位结点的hash算法简化会带来弊端,Hash冲突加剧,因此在链表节点数量大于8时，会将链表转化为红黑树进行存储。

 **5.查询时间复杂度**：从原来的遍历链表O(n)，变成遍历红黑树O(logN)。



#### 使用读写分离锁替代独占锁

顾名思义, 用ReadWriteLock将读写的锁分离开来, 尤其在**读多写少**的场合, 可以有效提升系统的并发能力.

ReadWriteLock是JDK5中提供的读写分离锁。读写分离锁可以有效地帮助减少锁竞争，以提升系统的性能。用锁分离的机制来提升性能非常容易理解，比如线程A1、A2、A3进行写操作，B1、B2、B3进行读操作，如果使用重入锁或者内部锁，则理论上说所有读之间，读与写之间、写与写之间都是串行操作。当B1进行读取时，B2、B3则需要等待锁。由于读操作并不会对数据的完整性造成破坏，这种等待显然是不合理的。因此，读写锁就有了发挥功能的余地。

在这种情况下，读写锁允许多个线程同时读，使得B1、B2、B3之间真正的并行。但是考虑到数据的完整性，写写操作和读写操作之间依然是需要相互等待和持有锁的。总的来说读写锁的访问约束如下：

- 读-读不互斥：读读之间不阻塞。
- 读-写互斥：读阻塞写，写也会阻塞读。
- 写-写互斥：写写阻塞。

演示代码如下：

```java
public class ReadWriteLockExample {

    private final static ReentrantReadWriteLock READ_WRITE_LOCK = new ReentrantReadWriteLock();

    private final static Lock READ_LOCK = READ_WRITE_LOCK.readLock();
    private final static Lock WRITE_LOCK = READ_WRITE_LOCK.writeLock();

    private int value = 0;

    public Object handleRead(Lock lock) throws InterruptedException {
        try {
            lock.lock();
            Thread.sleep(1000);
            System.out.println("reading value:" + value);
            return value;
        } finally {
            lock.unlock();
        }
    }

    public void handleWrite(Lock lock, int tmp) throws InterruptedException {
        try {
            lock.lock();
            Thread.sleep(1000);
            System.out.println("writing value:" + tmp);
            value = tmp;
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        final ReadWriteLockExample rwl = new ReadWriteLockExample();
        Thread readThread = new Thread(new Runnable() {

            @Override
            public void run() {
                try {
                    rwl.handleRead(READ_LOCK);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        Thread writeThread = new Thread(new Runnable() {

            @Override
            public void run() {
                try {
                    rwl.handleWrite(WRITE_LOCK, new Random().nextInt());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        for (int i = 0; i < 2; i++) {
            new Thread(writeThread).start();
        }

        for (int i = 0; i < 18; i++) {
            new Thread(readThread).start();
        }
    }
}
```

##### 读写分离锁ReentrantReadWriteLock详解

###### 一、性质

1. 可重入

   ​	可重入就是同一个线程持有该锁时，可以重复加锁。每次加锁的时候count值加1，每次释放锁的时候count减1，直到count为0，其他的线程才可以再次获取。

2. 读写分离

   ​	在多个线程操作同一数据时，如果多个线程全是读操作，那么这个多线程即使不加锁也不会出现任何问题。但是多个线程对同一数据进行写操作，如果不加锁则可能导致数据不一致的问题。所以为了提高程序效率，把写数据操作和读数据操作分开，加上两把不同的锁，不仅保证数据正确性，还能提高效率。

   ​	**读锁是多线程共享，写锁是单线程独占的。**

3. 支持可以锁降级

   ​	线程获取写入锁后可以获取读取锁，然后释放写入锁，这样就从写入锁变成了读取锁，从而实现锁降级的特性。在释放写锁之前获取读锁是为了保证数据的可见性，因为如果获取读锁在释放写锁之后的话，写锁被其他线程获取，改写了数据，在当前线程获取读锁后读到数据则是修改后的数据。遵循锁降级的步骤，则会阻塞其他线程获取写锁，直到当前线程使用数据并释放读锁后，其他线程才能获取写锁进行数据操作。

   ![img](https://pics5.baidu.com/feed/caef76094b36acafd16bfaad109d3f1500e99ca6.jpeg?token=508404373d6dae2420a3853483eca6c2&s=04B86832CE83F10106789CC10000E0B2)

4. 不支持锁升级

   ​	线程获取读锁是不能直接升级为写入锁的。需要释放所有读取锁，才可获取写锁。

   ![img](https://pics2.baidu.com/feed/32fa828ba61ea8d37a664cddfb4e824b241f5868.jpeg?token=96aeba47bbee024ee573ee242e0d6db4&s=61BC387284B849804CD4D1DF000050B2)

   **为什么不支持锁升级？**

   ​	读写锁的特点是如果线程都**申请读锁，是可以多个线程同时持有的，可是如果是写锁，只能有一个线程持有**，并且不可能存在读锁（先）和写锁（后）同时持有的情况。

   ​	正是因为不可能有读锁和写锁同时持有的情况，所以**升级写锁的过程中，需要等到所有的读锁都释放，此时才能进行升级。**

   ​	假设有 A，B 和 C 三个线程，它们都已持有读锁。假设线程 A 尝试从读锁升级到写锁。那么它必须等待 B 和 C 释放掉已经获取到的读锁。如果随着时间推移，B 和 C 逐渐释放了它们的读锁，此时线程 A 确实是可以成功升级并获取写锁。

   ​	但是我们考虑一种特殊情况。假设线程 A 和 B 都想升级到写锁，那么对于线程 A 而言，它需要等待其他所有线程，包括线程 B 在内释放读锁。而线程 B 也需要等待所有的线程，包括线程 A 释放读锁。这就是一种非常典型的死锁的情况。谁都愿不愿意率先释放掉自己手中的锁。（写-读互斥）

    	但是读写锁的升级并不是不可能的，也有可以实现的方案，如果我们保证每次只有一个线程可以升级，那么就可以保证线程安全。只不过最常见的 ReentrantReadWriteLock 对此并不支持。

   实例代码：
   
   ```java
   public class ReadWriteLockExample {
   
       private final static ReentrantReadWriteLock READ_WRITE_LOCK = new ReentrantReadWriteLock();
   
       private final static Lock READ_LOCK = READ_WRITE_LOCK.readLock();
       private final static Lock WRITE_LOCK = READ_WRITE_LOCK.writeLock();
   
       private int value = 0;
   
       public Object handleRead(Lock lock) throws InterruptedException {
           try {
               lock.lock();
               System.out.println(
                       "after readLock" + Thread.currentThread().getName() + "  time:" + System.currentTimeMillis());
               Thread.sleep(1000);
               return value;
           } finally {
               lock.unlock();
               System.out.println("readLock unlock" + Thread.currentThread().getName());
           }
       }
   
       public void handleWrite(Lock lock, int tmp) throws InterruptedException {
           try {
               lock.lock();
               System.out.println("writeLock time :" + System.currentTimeMillis());
               Thread.sleep(1000);
               System.out.println("writing value:" + tmp);
               value = tmp;
           } finally {
               lock.unlock();
           }
       }
   
       public static void main(String[] args) {
           final ReadWriteLockExample rwl = new ReadWriteLockExample();
           Thread readThread = new Thread(new Runnable() {
   
               @Override
               public void run() {
                   try {
                       rwl.handleRead(READ_LOCK);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
           });
   
           Thread writeThread = new Thread(new Runnable() {
   
               @Override
               public void run() {
                   try {
                       rwl.handleWrite(WRITE_LOCK, new Random().nextInt());
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
               }
           });
   
           for (int i = 0; i < 2; i++) {
               new Thread(writeThread).start();
           }
   
           for (int i = 0; i < 10; i++) {
               new Thread(readThread).start();
           }
       }
   }
   //writeLock time :1593568790864
   //writing value:-1608193388
   //after readLockThread-4  time:1593568791917
   //after readLockThread-6  time:1593568791920
   //.....
   //readLock unlockThread-4
   //readLock unlockThread-6
   //.....
   //writeLock time :1593568792938
   //writing value:-1524927570
   //after readLockThread-5  time:1593568793939
   //after readLockThread-7  time:1593568793940
   //.....
   //readLock unlockThread-5
   //readLock unlockThread-7
   //.....
   ```
   
   第二个写锁在获取时，需要等待已获取读锁的线程释放读锁才能获取写锁。

###### 二、使用

1. 基本使用

   写方法和读方法分别使用写锁和写锁。

   使用ReentrantReadWriteLock类实例化一个可重入锁对象，使用readLock和writeLock方法获取读写锁对象。读锁类和写锁类是ReentrantReadWriteLock类的静态内部类，读锁获取锁对象是共享锁，写锁获取锁对象是独占锁。

   ReentrantReadWriteLock需要注意的是在加锁后需要手动释放锁。

2. 锁升级

   读锁在获取写锁之前必须先释放读锁，这种设计可以实现在多线程情况下也可以锁升级。

   ```java
   public class CachedData {
    
       Object data;
       volatile boolean cacheValid;
       final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
    
       void processCachedData() {
           rwl.readLock().lock();
           if (!cacheValid) {
               //在获取写锁之前，必须首先释放读锁。
               rwl.readLock().unlock();
               rwl.writeLock().lock();
               try {
                   //这里需要再次判断数据的有效性,因为在我们释放读锁和获取写锁的空隙之内，可能有其他线程修改了数据。
                   if (!cacheValid) {
                       data = new Object();
                       cacheValid = true;
                   }
                   //在不释放写锁的情况下，直接获取读锁，这就是读写锁的降级。
                   rwl.readLock().lock();
               } finally {
                   //释放了写锁，但是依然持有读锁
                   rwl.writeLock().unlock();
               }
           }
    
           try {
               System.out.println(data);
           } finally {
               //释放读锁
               rwl.readLock().unlock();
           }
       }
   }
   ```

3. 其他方法

    暂置，源码研究其他方法，读写锁源码与ReentrantLock类似，特性相近。

#### 锁分离

在读写锁的思想上做进一步的延伸, 根据不同的功能拆分不同的锁, 进行有效的锁分离。

一个典型的示例便是LinkedBlockingQueue,在它内部, take和put操作本身是隔离的，

有若干个元素的时候, 一个在queue的头部操作, 一个在queue的尾部操作, 因此分别持有一把独立的锁。

[![img](https://images2018.cnblogs.com/blog/1285727/201806/1285727-20180613070619982-1230258202.png)](https://images2018.cnblogs.com/blog/1285727/201806/1285727-20180613070619982-1230258202.png)

```java
/** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
```

LinkedBlockingQueue详解见第七章 J.U.C中的源码分析。

#### 锁粗化



### 使用ThreadLocal

#### 使用示例

除了控制有限资源访问外, 我们还可以增加资源来保证对象线程安全.

对于一些线程不安全的对象, 例如SimpleDateFormat, 与其加锁让100个线程来竞争获取, 

不如准备100个SimpleDateFormat, 每个线程各自为营, 很快的完成format工作.

```java
public class ThreadLocalDemo {

    public static ThreadLocal<SimpleDateFormat> threadLocal = new ThreadLocal();

    public static void main(String[] args){
        ExecutorService service = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 100; i++) {
            service.submit(new Runnable() {
                @Override
                public void run() {
                    if (threadLocal.get() == null) {
                        threadLocal.set(new SimpleDateFormat("yyyy-MM-dd"));
                    }

                    System.out.println(threadLocal.get().format(new Date()));
                }
            });
        }
    }
}
```

#### 原理

关于ThreadLocal中set（value）方法，先获取当前线程对象，然后根据该线程对象获取当前线程的ThreadLocalMap对象，如果map非空则根据ThreadLocal-value键值对存放数据。

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

关于ThreadLocal的get（）方法，同样根据当前线程对象获取ThreadLocalMap对象。判断map是否为空，若为空则初始化；若不为空，则判断该ThreadLocal对象在map中是否有值，若没有值则初始化，若有值则返回该值。

```java
	public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```



#### 内存释放

- 手动释放: 调用threadlocal.set(null)或者threadlocal.remove()即可
- 自动释放: 关闭线程池, 线程结束后, 自动释放threadlocalmap.

```java
 1 public class StaticThreadLocalTest {
 2 
 3     private static ThreadLocal tt = new ThreadLocal();
 4     public static void main(String[] args) throws InterruptedException {
 5         ExecutorService service = Executors.newFixedThreadPool(1);
 6         for (int i = 0; i < 3; i++) {
 7             service.submit(new Runnable() {
 8                 @Override
 9                 public void run() {
10                     BigMemoryObject oo = new BigMemoryObject();
11                     tt.set(oo);
12                     // 做些其他事情
13                     // 释放方式一: 手动置null
14 //                    tt.set(null);
15                     // 释放方式二: 手动remove
16 //                    tt.remove();
17                 }
18             });
19         }
24         // 释放方式三: 关闭线程或者线程池
25         // 直接new Thread().start()的场景, 会在run结束后自动销毁线程
26 //        service.shutdown();
27 
28         while (true) {
29             Thread.sleep(24 * 3600 * 1000);
30         }
31     }
32 
33 }
34 // 构建一个大内存对象, 便于观察内存波动.
35 class BigMemoryObject{
36 
37     List<Integer> list = new ArrayList<>();
38 
39     BigMemoryObject() {
40         for (int i = 0; i < 10000000; i++) {
41             list.add(i);
42         }
43     }
44 }
```



#### 内存泄露

内存泄露主要出现在无法关闭的线程中， 例如web容器提供的并发线程池， 线程都是复用的。

由于ThreadLocalMap生命周期和线程生命周期一样长。对于一些被强引用持有的ThreadLocal， 如定义为static。

如果在使用结束后， 没有手动释放ThreadLocal，由于线程会被重复使用， 那么会出现之前的线程对象残留问题，造成内存泄露， 甚至业务逻辑紊乱。

对于没有强引用持有的ThreadLocal, 如方法内变量，是不是就万事大吉了呢？答案是否定的。

虽然ThreadLocalMap会在get和set等操作里删除key 为 null的对象，但是这个方法并不是100%会执行到。

看ThreadLocalMap源码即可发现， 只有调用了getEntryAfterMiss后才会执行清除操作，如果后续线程没满足条件或者都没执行get set操作，那么依然存在内存残留问题。

不管threadlocal是static还是非static的，都要像加锁解锁一样， 每次用完后， 手动清理，释放对象。

### 使用无锁操作

与锁相比, 使用CAS操作, 由于其非阻塞性, 因此不存在死锁问题, 同时线程之间的相互影响, 

也远小于锁的方式. 使用无锁的方案, 可以减少锁竞争以及线程频繁调度带来的系统开销.

> 例如生产消费者模型中, 可以使用BlockingQueue来作为内存缓冲区, 但他是基于锁和阻塞实现的线程同步.
>
> 如果想要在高并发场合下获取更好的性能, 则可以使用基于CAS的ConcurrentLinkedQueue. 
>
> 同理, 如果可以使用CAS方式实现整个生产消费者模型, 那么也将获得可观的性能提升, 如Disruptor框架.

## 十二、多线程开发的良好习惯

- 获取单例对象时需要保证线程安全，其中的方法也要保证线程安全。说明：资源驱动类、工具类、单例工厂类都需要注意。

- 给线程起个有意义的名字，这样可以方便找 Bug。
- 缩小同步范围，从而减少锁争用。例如对于 synchronized，应该尽量使用同步块而不是同步方法。
- 多用同步工具少用 wait() 和 notify()。首先，CountDownLatch, CyclicBarrier,  Semaphore 和 Exchanger 这些同步类简化了编码操作，而用 wait() 和 notify()  很难实现复杂控制流；其次，这些同步类是由最好的企业编写和维护，在后续的 JDK 中还会不断优化和完善。
- 使用 BlockingQueue 实现生产者消费者问题。
- 多用并发集合少用同步集合，例如应该使用 ConcurrentHashMap 而不是 Hashtable。
- 使用本地变量和不可变类来保证线程安全。
- 使用线程池而不是直接创建线程，这是因为创建线程代价很高，线程池可以有效地利用有限的线程来启动任务。

## 参考

[Java并发编程实战系列15之原子遍历与非阻塞同步机制(Atomic Variables and Non-blocking Synchronization)]: https://cloud.tencent.com/developer/article/1113519

[深入理解java内存模型]: https://www.jianshu.com/p/15106e9c4bf3

[认真的讲一讲：自旋锁到底是什么]: https://www.jianshu.com/p/9d3660ad4358

[深入理解Java中的逃逸分析]: https://blog.csdn.net/hollis_chuang/article/details/80922794
[对象和数组并不是都在堆上分配内存的]: https://www.hollischuang.com/archives/2398

[彻底搞清楚ConcurrentHashMap的实现原理]: https://youzhixueyuan.com/concurrenthashmap.html

[可重入读写锁ReentrantReadWriteLock的使用详解]: https://baijiahao.baidu.com/s?id=1649257642735492589&amp;wfr=spider&amp;for=pc
[ReentrantReadWriteLock——读写锁如何升级，为何读写锁不能插队？]: https://blog.csdn.net/zhangkaixuan456/article/details/107019540/

