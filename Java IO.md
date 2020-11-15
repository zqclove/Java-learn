# Java I/O

# 概览

Java中的 I/O 大概可以分为一下几种：

- 磁盘操作：`File`（从1.7开始可以使用 `Paths` 和 `Files` 代替 `File`）
- 字节操作：`InputStream` 和 `OutputStream`
- 字符操作：`Reader` 和 `Writer`
- 对象操作：`Serializable`
- 网络操作：`Socket`
- 新的输出\输入：NIO

其中`XXXStream`和`Reader`是抽象组件，使用时可以根据具体组件实现的方法使用。

# 磁盘操作

`File` 类可以用于表示文件和目录的信息，但是它**不表示文件的内容**。

递归地列出一个目录下所有文件：

```java
public static void listAllFiles(File dir) {
    if (dir == null || !dir.exists()) {
        return;
    }
    // 判断是否为文件
    if (dir.isFile()) {
        System.out.println(dir.getName());
        return;
    }
    // 是目录则递归遍历该目录下所有目录或文件
    for (File file : dir.listFiles()) {
        listAllFiles(file);
    }
}
```

# 字节操作

`XXXStream`是所有输入流\输出流的父类，是一个抽象类，定义了所有输入流\输出流的共同特征

## 实现文件复制

```java
public static void copyFile(String src, String dist) throws IOException {
    FileInputStream in = new FileInputStream(src);
    FileOutputStream out = new FileOutputStream(dist);

    byte[] buffer = new byte[20 * 1024];
    int cnt;

    // read() 最多读取 buffer.length 个字节
    // 返回的是实际读取的个数
    // 返回 -1 的时候表示读到 eof，即文件尾
    while ((cnt = in.read(buffer, 0, buffer.length)) != -1) {
        out.write(buffer, 0, cnt);
    }

    in.close();
    out.close();
}
```

## 装饰者模式

Java I/O 使用了装饰者模式来实现。以 `InputStream` 为例，

- `InputStream` 是抽象组件；
- `FileInputStream` 是 `InputStream` 的子类，属于具体组件，提供了字节流的输入操作；
- `FilterInputStream` 属于抽象装饰者，装饰者用于装饰组件，为组件提供额外的功能。例如 `BufferedInputStream` 为 `FileInputStream` 提供缓存的功能。

实例化一个具有缓存功能的字节流对象时，只需要在 `FileInputStream` 对象上再套一层 `BufferedInputStream `对象即可。

```java
FileInputStream fileInputStream = new FileInputStream(filePath);
BufferedInputStream bufferedInputStream = new BufferedInputStream(fileInputStream);
```

`DataInputStream` 装饰者提供了对更多数据类型进行输入的操作，比如 int、double 等基本类型。

# 字符操作

`Reader`是基于字符操作的输入流抽象类，`Writer`是基于字符操作输出流抽象类。

## 编码与解码

**编码就是把字符转换为字节，而解码是把字节重新组合成字符。**

如果编码和解码过程使用不同的编码方式那么就出现了乱码。

- GBK 编码中，中文字符占 2 个字节，英文字符占 1 个字节；
- UTF-8 编码中，中文字符占 3 个字节，英文字符占 1 个字节；
- UTF-16be 编码中，中文字符和英文字符都占 2 个字节。

UTF-16be 中的 be 指的是 Big Endian，也就是大端。相应地也有 UTF-16le，le 指的是 Little Endian，也就是小端。

​	Java 的内存编码使用双字节编码 UTF-16be，这不是指 Java 只支持这一种编码方式，而是说 **`char` 这种类型使用  UTF-16be 进行编码。**`char` 类型占 16 位，也就是两个字节，Java 使用这种双字节编码是为了让一个中文或者一个英文都能使用一个  `char` 来存储。

## String的编码方式

`	String` 可以看成一个字符序列（`String`使用`char`数组存储数据——1.8，1.9之后使用`byte`数组），可以指定一个编码方式将它编码为字节序列，也可以指定一个编码方式将一个字节序列解码为 `String`。

```java
String str1 = "中文";
byte[] bytes = str1.getBytes("UTF-8");
String str2 = new String(bytes, "UTF-8");
System.out.println(str2);
```

​	在调用无参数 getBytes() 方法时，默认的编码方式不是 UTF-16be。双字节编码的好处是可以使用一个 char  存储中文和英文，而将 String 转为 bytes[] 字节数组就不再需要这个好处，因此也就不再需要双字节编码。getBytes()  的默认编码方式与平台有关，一般为 UTF-8。

```java
byte[] bytes = str1.getBytes();
```

## Reader 与 Writer

不管是磁盘还是网络传输，最小的存储单元都是字节，而不是字符。但是在程序中操作的通常是字符形式的数据，因此需要提供对字符进行操作的方法。

- `InputStreamReader` 实现从字节流解码成字符流。（解码）
- `OutputStreamWriter` 实现字符流编码成为字节流。（编码）

## 实现逐行输出文本文件的内容

```java
public static void readFileContent(String filePath) throws IOException {

    FileReader fileReader = new FileReader(filePath);
    BufferedReader bufferedReader = new BufferedReader(fileReader);

    String line;
    while ((line = bufferedReader.readLine()) != null) {
        System.out.println(line);
    }

    // 装饰者模式使得 BufferedReader 组合了一个 Reader 对象
    // 在调用 BufferedReader 的 close() 方法时会去调用 Reader 的 close() 方法
    // 因此只要一个 close() 调用即可
    bufferedReader.close();
}
```

# 对象操作

## 序列化

序列化就是将一个对象转换成字节序列，方便存储和传输。

- 序列化：ObjectOutputStream.writeObject()
- 反序列化：ObjectInputStream.readObject()

**不会对静态变量进行序列化，因为序列化只是保存对象的状态，静态变量属于类的状态。**

## Serializable

序列化的类需要实现 Serializable 接口，它只是一个标准，没有任何方法需要实现，但是如果不去实现它的话而进行序列化，会抛出异常。

```java
public static void main(String[] args) throws IOException, ClassNotFoundException {

    A a1 = new A(123, "abc");
    String objectFile = "file/a1";

    ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(objectFile));
    objectOutputStream.writeObject(a1);
    objectOutputStream.close();

    ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(objectFile));
    A a2 = (A) objectInputStream.readObject();
    objectInputStream.close();
    System.out.println(a2);
}

private static class A implements Serializable {

    private int x;
    private String y;

    A(int x, String y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public String toString() {
        return "x = " + x + "  " + "y = " + y;
    }
}
```

## transient

transient 关键字可以使一些属性不会被序列化。

ArrayList 中存储数据的数组 elementData 是用 transient 修饰的，因为这个数组是动态扩展的，并不是所有的空间都被使用，因此就不需要所有的内容都被序列化。通过重写序列化和反序列化方法，使得可以只序列化数组中有内容的那部分数据。

```java
private transient Object[] elementData;
```

# 网络操作

Java 中的网络支持：

- InetAddress：用于表示网络上的硬件资源，即 IP 地址；
- URL：统一资源定位符；
- Sockets：使用 TCP 协议实现网络通信；
- Datagram：使用 UDP 协议实现网络通信。

## InetAddress

没有公有的构造函数，只能通过静态方法来创建实例。

```java
InetAddress.getByName(String host);
InetAddress.getByAddress(byte[] address);
```

## URL

可以直接从 URL 中读取字节流数据。

```java
public static void main(String[] args) throws IOException {

    URL url = new URL("http://www.baidu.com");

    /* 字节流 */
    InputStream is = url.openStream();

    /* 字符流 */
    InputStreamReader isr = new InputStreamReader(is, "utf-8");

    /* 提供缓存功能 */
    BufferedReader br = new BufferedReader(isr);

    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }

    br.close();
}
```

## Sockets

​	Sockets是使用TCP协议进行网络通信的。所谓socket 通常也称作”套接字“，用于描述IP地址和端口，是一个通信链的句柄。应用程序通常通过”套接字”向网络发出请求或者应答网络请求。

- `ServerSocket`：服务器端类
- `Socket`：客户端类（服务端也有）
- 服务器和客户端通过 `InputStream`和 `OutputStream` （流）进行输入输出。

![Sockets工作方式](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\Sockets工作方式.png)

### Socket编程

**服务器端套路**

1. 创建`ServerSocket`对象，绑定监听端口。
2. 通过accept（）方法监听客户端请求。
3. 连接建立后，通过输入流（`InputStream`）读取客户端发送的请求信息。
4. 通过输出流（`OutputSteam`）向客户端发送响应信息。
5. 关闭响应的资源。

**客户端套路**

1. 创建Socket对象，指明需要连接的服务器的地址和端口号。
2. 连接建立后，通过输出流（`OutputStream`）向服务器发送请求信息。
3. 通过输入流获取服务器响应的信息。
4. 关闭相应资源。（只有客户端关闭响应资源或以某种约定，服务端才能知道客户端消息发送完毕而不继续等待）

**多线程实现服务器与多客户端之间通信步骤**

1. 服务器端创建`ServerSocket`，循环调用accept（）等待客户端连接。
2. 客户端创建一个`socket`并请求和服务器端连接。
3. 服务器端接受客户端请求，创建`socket`与该客户建立专线连接。
4. 建立连接的两个`socket`在一个单独的线程上对话。
5. 服务器端继续等待新的连接。




​	在Socket编程中，主要涉及两个角色：客户端和服务端。

​	客户端与服务端的Socket编程大致上类似，但又存在不同。服务端主要等待客户端连接；客户端主要发起请求。客户端和服务端可以建立**双向通信**，既可以发送消息也可以接受消息。服务端可以通过**多线程技术**实现与多个客户端连接。

​	对于通信的中消息结束标志，可以通过指定消息长度的方式（消息包含长度+类型+数据）告知对方结束位置。

### 参考资料

[Java 网络编程 之 socket 的用法与实现]: https://blog.csdn.net/a78270528/article/details/80318571

## Datagram

- DatagramSocket：通信类
- DatagramPacket：数据包类

# NIO



# IO模型

一个IO操作通常包括两个阶段：

1. 等待数据准备好（数据准备）；
2. 从内核向进程复制数据；

​	对于一个套接字上的输入操作，**第一步**通常涉及等待数据从网络中到达。到所等待数据到达时，它被复制到内核中某个缓冲区。**第二步**就是把数据从内核缓冲区复制到应用进程缓冲区。

Unix 有五种 I/O 模型：

- 阻塞式 I/O（同步阻塞式IO）
- 非阻塞式 I/O（同步非阻塞式IO）
- I/O 复用（select 和 poll）
- 信号驱动式 I/O（SIGIO）
- 异步 I/O（AIO）

​	同步和异步在IO模型中的区别主要在IO操作的第二个阶段（从内核向进程复制数据）。**同步**的方式是进程会阻塞，主动从内核缓冲中获取数据；**异步**的方式是不阻塞进程，由CPU将内核缓冲区中的数据复制到进程缓冲区中。两者的区别就在于CPU参与度，也就是IO操作是否全部由内核来操作。

​	阻塞式和非阻塞式区别在于第一阶段，数据准备过程中阻塞式IO的进程不会继续执行，而非阻塞式IO的进程可以继续执行。

​	值得一提的是，异步IO是不存在阻塞的情况的，后面会详细说明。

## 同步阻塞式IO

​	应用进程被阻塞，直到数据从内核缓冲区复制到应用进程缓冲区中才返回。

​	应该注意到，在阻塞的过程中，其它应用进程还可以执行，因此阻塞不意味着整个操作系统都被阻塞。因为其它应用进程还可以执行，所以不消耗 CPU 时间，这种模型的 CPU 利用率会比较高。

下图中，**recvfrom()** 用于接收 Socket 传来的数据，并复制到应用进程的缓冲区 **buf** 中。这里把 **recvfrom()** 当成系统调用。

```c
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, 
                 struct sockaddr *src_addr, socklen_t *addrlen);
```

![阻塞式IO模型](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\阻塞式IO模型.png)

## 同步非阻塞式IO

​	应用进程执行系统调用之后，内核返回一个错误码。应用进程可以继续执行，但是需要不断的执行系统调用来获知 I/O 是否完成，这种方式称为**轮询（polling）**。

​	**由于 CPU 要处理更多的系统调用，因此这种模型的 CPU 利用率比较低。**

![非阻塞式IO模型](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\非阻塞式IO模型.png)

## 异步IO

​	应用进程执行 `aio_read` 系统调用会立即返回，应用进程可以继续执行，不会被阻塞，内核会在所有操作完成之后向应用进程发送信号（包括数据从内核缓冲区拷贝到用户缓冲区的过程）。

​	**异步 I/O 与信号驱动 I/O 的区别在于，异步 I/O 的信号是通知应用进程 I/O 完成，而信号驱动 I/O 的信号是通知应用进程可以开始 I/O。**

![异步IO模型](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\异步IO模型.png)

## IO复用

​	使用 `select` 或者 `poll` 等待数据，并且可以等待多个套接字中的任何一个变为可读。这一过程会被阻塞，当某一个套接字可读时返回，之后再使用 `recvfrom` 把数据从内核复制到进程中。

​	关于`select`、`poll`和`epoll`后面详细介绍。

​	它可以让单个进程具有处理多个 I/O 事件的能力。又被称为 **Event Driven I/O，即事件驱动 I/O。**

​	如果一个 Web 服务器没有 I/O 复用，那么每一个 Socket 连接都需要创建一个线程去处理。如果同时有几万个连接，那么就需要创建相同数量的线程。相比于多进程和多线程技术，I/O 复用不需要进程线程创建和切换的开销，系统开销更小。

![IO复用模型](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\IO复用模型.png)

## 信号驱动式IO

​	应用进程使用 `sigaction` 系统调用，内核立即返回，应用进程可以继续执行，也就是说等待数据阶段应用进程是非阻塞的。内核在数据到达时向应用进程发送 **SIGIO 信号**，应用进程收到之后在信号处理程序中调用 `recvfrom` 将数据从内核复制到应用进程中。

​	**相比于非阻塞式 I/O 的轮询方式，信号驱动 I/O 的 CPU 利用率更高。**

![信号驱动式IO模型](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\信号驱动式IO模型.png)

## 五大 I/O 模型比较

- 同步 I/O：将数据从内核缓冲区复制到应用进程缓冲区的阶段（第二阶段），应用进程会阻塞。
- 异步 I/O：第二阶段应用进程不会阻塞。

**同步 I/O 包括阻塞式 I/O、非阻塞式 I/O、I/O 复用和信号驱动 I/O** ，它们的主要区别在第一个阶段。

非阻塞式 I/O 、信号驱动 I/O 和异步 I/O 在第一阶段不会阻塞。

![五大IO模型比较图](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\五大IO模型比较图.png)

# IO复用

​	`select`/`poll`/`epoll` 都是 I/O 多路复用的具体实现，`select` 出现的最早，之后是 `poll`，再是 `epoll`。

## select

​	`select`允许应用程序监视一组文件描述符，**等待一个或多个描述符成为就绪状态**，从而完成IO操作。`select`目前几乎在所有的平台上支持，其良好跨平台支持是它的一个优点。`select`缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但是这样会造成效率降低。

```C
int select(int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, 
           struct timeval *timeout);
```

​	**fd_set**使用数组实现，数组大小使用**FD_SETSIZE**定义，所以只能监听少于其数量的描述符（数量限制）。 

​	由上面函数可以得知，`select`函数监视的文件描述符分为3类：**readfds**、**writefds**、**exceptfds**，分别对应**读**、**写**、**异常条件**的描述符集合。**timeout**为超时参数，调用`select`会一直阻塞直到有描述符的事件达到或者等待的时间超过**timeout**（为null则立即返回）。

​	成功调用返回结果大于 0，出错返回结果为 -1，超时返回结果为 0。当`select`函数返回后，可以通过遍历**fd_set**来找到就绪的描述符。

​	下面是使用`select`的简单例子：

```c
fd_set fd_in, fd_out;
struct timeval tv;

// Reset the sets
FD_ZERO( &fd_in );
FD_ZERO( &fd_out );

// Monitor sock1 for input events
FD_SET( sock1, &fd_in );

// Monitor sock2 for output events
FD_SET( sock2, &fd_out );

// Find out which socket has the largest numeric value as select requires it
int largest_sock = sock1 > sock2 ? sock1 : sock2; 
// 在read和write两个socket中获取最大数量作为调用select参数时的数量，有最大限制

// Wait up to 10 seconds
tv.tv_sec = 10;
tv.tv_usec = 0;

// Call the select
int ret = select( largest_sock + 1, &fd_in, &fd_out, NULL, &tv );

// Check if select actually succeed
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
	//调用成功，遍历每个fd_set获取就绪的描述符 
	
    if ( FD_ISSET( sock1, &fd_in ) )
        // input event on sock1

    if ( FD_ISSET( sock2, &fd_out ) )
        // output event on sock2
}
```



## poll

​	`poll`的功能与`select`类似，也是等待一组描述符中的一个成为就绪状态。

```c
int poll(struct pollfd *fds, unsigned int nfds, int timeout);
```

​	不同于`select`使用三个位图来表示三个**fd_set**的方式，`poll`使用一个`pollfd`的指针（数组）实现：

```c
struct pollfd {
	int   fd;         /* file descriptor */
	short events;     /* requested events */
	short revents;    /* returned events */
};
```

​	因为可以在程序中自定义`pollfd`数组的大小，所以使用`poll`没有数量限制。

​	调用成功返回结果大于0，出错返回结果为-1，超时返回结果为0。和`select`一样，`poll`返回后同样需要遍历`pollfd`数组来获取就绪的描述符。

​	下面是使用`poll`的简单例子：

```c
// The structure for two events
struct pollfd fds[2]; // 可以自定义数组大小，没有最大数量限制

// Monitor sock1 for input
fds[0].fd = sock1;
fds[0].events = POLLIN;

// Monitor sock2 for output
fds[1].fd = sock2;
fds[1].events = POLLOUT;

// Wait 10 seconds
int ret = poll( &fds, 2, 10000 );
// Check if poll actually succeed
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
    // 调用成功，遍历pollfd数组获取就绪的描述符。
    
    // If we detect the event, zero it out so we can reuse the structure
    if ( fds[0].revents & POLLIN )
        fds[0].revents = 0;
        // input event on sock1

    if ( fds[1].revents & POLLOUT )
        fds[1].revents = 0;
        // output event on sock2
}
```



## 比较select和poll

### 1. 功能

`select` 和 `poll` 的功能基本相同，不过在一些实现细节上有所不同：

- `select` 会修改描述符，而 `poll` 不会；
- `select` 的描述符类型使用数组实现，**FD_SETSIZE** 大小默认为 1024，因此默认只能监听少于 1024 个描述符。如果要监听更多描述符的话，需要修改 **FD_SETSIZE** 之后重新编译；而 `poll` 没有描述符数量的限制；
- `poll` 提供了更多的事件类型，并且对描述符的重复利用上比 `select` 高。
- 如果一个线程对某个描述符调用了 `select` 或者 `poll`，另一个线程关闭了该描述符，会导致调用结果不确定。

### 2. 速度

`select` 和 `poll` 速度都比较慢，每次调用都需要将全部描述符从应用进程缓冲区复制到内核缓冲区。

### 3. 可移植性

几乎所有的系统都支持 `select`，但是只有比较新的系统支持 `poll`。

## epoll



# 参考资料

[CyC2018之JavaIO]: https://github.com/CyC2018/CS-Notes/blob/master/notes/JavaIO.md

[同步、异步、阻塞、非阻塞IO总结（IO模型总结）]: https://blog.csdn.net/qq_36573828/article/details/89149057

[Linux IO模式及 select、poll、epoll详解]: https://segmentfault.com/a/1190000003063859

