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

​	NIO（Non-blocking I/O，在Java领域也被称为New I/O），是一种**同步非阻塞**的I/O模型，也是I/O多路复用的基础，已经被越来越多地应用到大型应用服务器，成为解决高并发与大量连接、I/O处理问题的有效方式。	

​	新的输入/输出 (NIO) 库是在 JDK 1.4 中引入的，弥补了原来的 I/O 的不足，提供了**高速的、面向块**的 I/O。

## 流与块

​	I/O 与 NIO 最重要的**区别是数据打包和传输的方式**，**I/O 以流的方式处理数据**，而 **NIO 以块的方式处理数据**。

​	**面向流的 I/O 一次处理一个字节数据**：一个输入流产生一个字节数据，一个输出流消费一个字节数据。为流式数据创建过滤器非常容易，链接几个过滤器，以便每个过滤器只负责复杂处理机制的一部分。不利的一面是，面向流的 I/O 通常相当慢。

​	**面向块的 I/O 一次处理一个数据块**，按块处理数据比按流处理数据要快得多。但是面向块的 I/O 缺少一些面向流的 I/O 所具有的优雅性和简单性。

​	I/O 包和 NIO 已经很好地集成了，`java.io.*` 已经以 NIO 为基础重新实现了，所以现在它可以利用 NIO 的一些特性。例如，`java.io.*` 包中的一些类包含以块的形式读写数据的方法，这使得即使在面向流的系统中，处理速度也会更快。

## 通道与缓冲区

### 通道

​	**通道 `Channel` 是对原 I/O 包中的流的模拟，可以通过它读取和写入数据**。

​	通道与流的不同之处在于，**流只能在一个方向上移动**(一个流必须是 InputStream 或者 OutputStream 的子类)，而**通道是双向的，可以用于读、写或者同时用于读写**。

​	通道包括以下类型：

- **FileChannel**：从文件中读写数据；
- **DatagramChannel**：通过 UDP 读写网络中数据；
- **SocketChannel**：通过 TCP 读写网络中数据；
- **ServerSocketChannel**：可以**监听**新进来的 TCP 连接，对每一个新进来的连接都会创建一个 **SocketChannel**。

### 缓冲区

​	发送给一个通道的所有数据都必须首先放到缓冲区中，同样地，从通道中读取的任何数据都要先读到缓冲区中。也就是说，**不会直接对通道进行读写数据，而是要先经过缓冲区**。

​	缓冲区实质上是一个数组，但它不仅仅是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。

缓冲区包括以下类型：

- **ByteBuffer**
- **CharBuffer**
- **ShortBuffer**
- **IntBuffer**
- **LongBuffer**
- **FloatBuffer**
- **DoubleBuffer**

## 缓冲区状态变量

- **capacity**：最大容量；
- **position**：当前已经读写的字节数；
- **limit**：还可以读写的字节数的最后位置；

**状态变量的改变过程举例**：

① 新建一个大小为 8 个字节的缓冲区，此时 **position** 为 0，而 **limit** = **capacity** = 8。**capacity** 变量不会改变，下面的讨论会忽略它。

![缓冲区状态变量1](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\缓冲区状态变量1.png)

② 从输入通道中读取 5 个字节数据写入缓冲区中，此时 **position** 为 5，**limit** 保持不变。

![缓冲区状态变量2](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\缓冲区状态变量2.png)

③ 在将缓冲区的数据写到输出通道之前，需要先调用 **flip()** 方法，这个方法将 **limit** 设置为当前 **position**，并将 **position** 设置为 0。

![缓冲区状态变量3](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\缓冲区状态变量3.png)

④ 从缓冲区中取 4 个字节到输出缓冲中，此时 **position** 设为 4。

![缓冲区状态变量4](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\缓冲区状态变量4.png)

⑤ 最后需要调用 **clear()** 方法来清空缓冲区，此时 **position** 和 **limit** 都被设置为最初位置。

![缓冲区状态变量5](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\缓冲区状态变量5.png)



## 选择器

​	NIO 常常被叫做**非阻塞 IO**，主要是因为 NIO 在网络通信中的非阻塞特性被广泛使用。

​	NIO 实现了 IO 多路复用中的 **Reactor** 模型，**一个线程 Thread** 使用**一个选择器 Selector** 通过**轮询**的方式去监听**多个通道 Channel 上的事件**，从而让一个线程就可以处理多个事件。

​	**通过配置监听的通道 Channel 为非阻塞**，那么当 Channel 上的 IO 事件还未到达时，就不会进入阻塞状态一直等待，而是继续轮询其它 Channel，找到 IO 事件已经到达的 Channel 执行。

​	因为创建和切换线程的开销很大，因此使用一个线程来处理多个事件而不是一个线程处理一个事件，对于 IO 密集型的应用具有很好地性能。

​	应该注意的是，**只有 SocketChannel 才能配置为非阻塞**，而 FileChannel 不能，为 FileChannel 配置非阻塞也没有意义。

![选择器](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\选择器.png)

下面介绍使用选择器应用在套接字上的编程模板：

1. **创建选择器**：

    选择器对象通过静态方法获取：

    ```java
    Selector selector = Selector.open();
    ```

2. **将通道注册到选择器上**：

    ```java
    ServerSocketChannel ssChannel = ServerSocketChannel.open();
    ssChannel.configureBlocking(false);
    ssChannel.register(selector, SelectionKey.OP_ACCEPT);
    ```

    ​	**通道必须配置为非阻塞模式**，否则使用选择器就没有任何意义了，因为如果通道在某个事件上被阻塞，那么服务器就不能响应其它事件，必须等待这个事件处理完毕才能去处理其它事件，显然这和选择器的作用背道而驰。

    ​	在将通道注册到选择器上时，还需要指定要注册的**具体事件**（类似`epoll`的**epoll_event**集合），主要有以下几类：

    - **SelectionKey.OP_CONNECT**
    - **SelectionKey.OP_ACCEPT**
    - **SelectionKey.OP_READ**
    - **SelectionKey.OP_WRITE**

    它们在 **SelectionKey** 的定义如下：

    ```java
    public static final int OP_READ = 1 << 0;
    public static final int OP_WRITE = 1 << 2;
    public static final int OP_CONNECT = 1 << 3;
    public static final int OP_ACCEPT = 1 << 4;
    ```

    可以看出每个事件可以被当成一个位域，从而组成事件集整数（用二进制位数上的1表示对某事件感兴趣）。例如：

    ```java
    int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
    // OP_READ = 00001 , OP_WRITE = 00100
    // 所以两者或运算结果为： 00101 ，该数就表示对读写感兴趣
    ```

3. **监听事件**：

    **个人问题**：关于**select()**的阻塞调用问题：使用**select()**接受请求，为什么还说NIO是非阻塞的呢？

    **答**：在NIO中，在处理请求的线程在接受请求（等待数据）的过程中，的确是阻塞的，但接受请求后，其相应工作可以派发给别的任务完成，处理请求的线程可以继续处理别的请求。（参考https://www.zhihu.com/question/64056856）

    **个人猜测**：这里的非阻塞与通道配置有关，在第2步的代码中可以看到会将通道设置成非阻塞，所以猜测说NIO非阻塞的原因是在通道上，即对单个事件的阻塞与否。

    ```java
    int num = selector.select();
    ```

    使用 **select()** 来监听到达的事件，它会**一直阻塞**直到有至少一个事件到达。

4. **获取到达的事件**：

    ```java
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = keys.iterator();
    while (keyIterator.hasNext()) {
        SelectionKey key = keyIterator.next();
        if (key.isAcceptable()) {
            // ...
        } else if (key.isReadable()) {
            // ...
        }
        keyIterator.remove();
    }
    ```

5. **事件循环**：

    因为一次 **select()** 调用不能处理完所有的事件，并且服务器端有可能需要一直监听事件，因此服务器端处理事件的代码一般会放在一个死循环内。

```java
while (true) {
    int num = selector.select();
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = keys.iterator();
    while (keyIterator.hasNext()) {
        SelectionKey key = keyIterator.next();
        if (key.isAcceptable()) {
            // ...
        } else if (key.isReadable()) {
            // ...
        }
        keyIterator.remove();
    }
}
```

## 套接字NIO实例

```java
public class NIOServer {

    public static void main(String[] args) throws IOException {

        Selector selector = Selector.open();

        ServerSocketChannel ssChannel = ServerSocketChannel.open();
        ssChannel.configureBlocking(false);
        ssChannel.register(selector, SelectionKey.OP_ACCEPT);

        ServerSocket serverSocket = ssChannel.socket();
        InetSocketAddress address = new InetSocketAddress("127.0.0.1", 8888);
        serverSocket.bind(address);

        while (true) {

            selector.select();
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = keys.iterator();

            while (keyIterator.hasNext()) {

                SelectionKey key = keyIterator.next();

                if (key.isAcceptable()) {

                    ServerSocketChannel ssChannel1 = 
                        (ServerSocketChannel) key.channel();

                    // 服务器会为每个新连接创建一个 SocketChannel
                    SocketChannel sChannel = ssChannel1.accept();
                    sChannel.configureBlocking(false);

                    // 这个新连接主要用于从客户端读取数据
                    sChannel.register(selector, SelectionKey.OP_READ);

                } else if (key.isReadable()) {

                    SocketChannel sChannel = (SocketChannel) key.channel();
                    System.out.println(readDataFromSocketChannel(sChannel));
                    sChannel.close();
                }

                keyIterator.remove();
            }
        }
    }

	private static String readDataFromSocketChannel(SocketChannel sChannel) 
        throws IOException {

        ByteBuffer buffer = ByteBuffer.allocate(1024);
        StringBuilder data = new StringBuilder();

        while (true) {

            buffer.clear();
            int n = sChannel.read(buffer);
            if (n == -1) {
                break;
            }
            buffer.flip();
            int limit = buffer.limit();
            char[] dst = new char[limit];
            for (int i = 0; i < limit; i++) {
                dst[i] = (char) buffer.get(i);
            }
            data.append(dst);
            buffer.clear();
        }
        return data.toString();
    }
}
public class NIOClient {

    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1", 8888);
        OutputStream out = socket.getOutputStream();
        String s = "hello world";
        out.write(s.getBytes());
        out.close();
    }
}
```

## 内存映射文件

​	内存映射文件 I/O 是一种读和写文件数据的方法，它可以比常规的基于流或者基于通道的 I/O 快得多。

​	向内存映射文件写入可能是危险的，只是改变数组的单个元素这样的简单操作，就可能会直接修改磁盘上的文件。修改数据与将数据保存到磁盘是没有分开的。

​	下面代码行将文件的前 1024 个字节映射到内存中，map() 方法返回一个 MappedByteBuffer，它是 ByteBuffer 的子类。因此，可以像使用其他任何 ByteBuffer 一样使用新映射的缓冲区，操作系统会在需要时负责执行映射。

```java
MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_WRITE, 0, 1024);
```

## 对比

NIO 与普通 I/O 的区别主要有以下两点：

- NIO 是非阻塞的；
- NIO 面向块，I/O 面向流。

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

​	它可以**让单个进程具有处理多个 I/O 事件的能力**。又被称为 **Event Driven I/O，即事件驱动 I/O。**

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

​	`epoll`是在2.6内核中提出的，是之前的`select`和`poll`的增强版本。相对于`select`和`poll`来说，`epoll`更加灵活，没有描述符数量限制。`epoll`使用**一个文件描述符管理多个描述符**，将用户关系的描述符的事件存放到内核的一个事件表中，这样在用户控件和内核空间的复制只需要一次。

​	`epoll` 仅适用于 Linux OS。

### epoll操作过程

​	`epoll`操作过程需要三个接口：

```c
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

1. **int epoll_create(int size)**：

    该接口会创建一个`epoll`句柄，其中参数**size**用来告诉内核这个监听的数目一共有多大。该参数不同于`select()`中的第一个参数，给出最大监听的fd+1的值。**size**参数并不是限制`epoll`所能监听的描述符最大个数，只是对内核初始分配数据结构的一个建议。

    当创建好`epoll`句柄后，它会占用一个 fd值。在linux下，如果查看/proc/进程id/fd/，是能够看到这个fd的。所以在使用完`epoll`后，必须调用`close()`关闭，否则可能导致fd被耗尽。

2. **int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)**：

    该接口是对指定描述符执行**op**操作。其中各参数含义为：

    - **epfd整型参数**：是**epoll_create()**函数的返回值；

    - **op整型参数**：表示op操作，用三个宏表示：**EPOLL_CTL_ADD（添加）**，**EPOLL_CTL_DEL（删除）**，**EPOLL_CTL_MOD（修改）**。分别添加、删除和修改对fd的监听事件；

    - **fd整型参数**：是需要监听的fd（外部socket）；

    - **event指针参数**：是告诉内核需要监听什么事件，**epoll_event**结构如下：

        ```c
        struct epoll_event {
         __unit32_t events; // 下面宏的集合，例如 events = EPOLLIN | EPOLLONESHOT
         epoll_data_t data; // 用户数据变量，好像是外部socket的连接
        };
        
        // events可以是以下宏的集合：
        EPOLLIN ：表示对应的文件描述符可以读（包括对端socket正常关闭）；
        EPOLLOUT：表示对应的文件描述符可以写；
        EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
        EPOLLERR：表示对应的文件描述符发生错误；
        EPOLLHUB：表示对应的文件描述符被挂断；
        EPOLLET ：将EPOLL设为边缘触发（Edge Triggerd）模式，
            这是相对于水平触发（Level Triggerd）来说的；
        EPOLLONESHOT：只监听一次事件。当监听完这次事件之后，如果还需要继续监听这个socket的话，
            需要再次把这个socket加入到EPOLL队列里；
        ```

3. **int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout)**：

    等待**epfd**上的IO事件，最多返回**maxevents**数量的事件。

    - **events**参数：用来**从内核得到事件的集合**（程序创建的数组，作为参数传给内核）；
    - **maxevents**参数：告诉内核这个**events**数组有多大，这个**maxevents**的值不能大于**epoll_create()**中的**size**的值；
    - **timeout**：参数是超时时间（毫秒，0会立即返回；-1将不确定，也可以说是永久阻塞）。

    **该函数返回需要处理的事件数目**，如返回0表示已超时。

------

​	**epoll_ctl()** 用于**向内核注册新的描述符或者是改变某个文件描述符的状态**。已注册的描述符在内核中会被维护在一棵红黑树上，通过回调函数内核会将 I/O 准备好的描述符加入到一个链表中管理，进程调用 **epoll_wait()** 便可以得到事件完成的描述符。

​	使用`epoll`的例子：

```c
// Create the epoll descriptor. Only one is needed per app, and is used to monitor all sockets.
// The function argument is ignored (it was not before, but now it is), so put your favorite number here
int pollingfd = epoll_create( 0xCAFE ); // 创建epoll句柄

if ( pollingfd < 0 )
 // report error

// Initialize the epoll structure in case more members are added in future
struct epoll_event ev = { 0 }; // 初始化事件

// Associate the connection class instance with the event. You can associate anything
// you want, epoll does not use this information. We store a connection class pointer, pConnection1
ev.data.ptr = pConnection1;

// Monitor for input, and do not automatically rearm the descriptor after the event
ev.events = EPOLLIN | EPOLLONESHOT; // 表明需要监听什么事件
// Add the descriptor into the monitoring list. We can do it even if another thread is
// waiting in epoll_wait - the descriptor will be properly added
if ( epoll_ctl( epollfd, EPOLL_CTL_ADD, pConnection1->getSocket(), &ev ) != 0 ) 
    // 向内核注册新的描述符，为0报错。
    // report error

// Wait for up to 20 events (assuming we have added maybe 200 sockets before that it may happen)
struct epoll_event pevents[ 20 ]; // 用于存放已到达的事件

// Wait for 10 seconds, and retrieve less than 20 epoll_event and store them into epoll_event array
int ready = epoll_wait( pollingfd, pevents, 20, 10000 );
// Check if epoll actually succeed
if ( ret == -1 )
    // report error and abort
else if ( ret == 0 )
    // timeout; no event detected
else
{
    // Check if any events detected
    for ( int i = 0; i < ready; i++ )
    {
        // 只操作事件EPOLLIN
        if ( pevents[i].events & EPOLLIN )
        {
            // Get back our connection pointer
            Connection * c = (Connection*) pevents[i].data.ptr;
            c->handleReadEvent();
         }
    }
}
```

------

**个人总结**：

​	操作`epoll`的三个接口，主要用于创建句柄、向内核注册或修改描述符、获取事件。通俗来讲就是，准备epoll了（创建句柄，**epoll_create()**）、epoll要干什么（注册并表明干什么和监听什么事件）、获取epoll结果（获取监听事件）。

​	`epoll`类似一个应用程序与内核之间的通道规范，应用程序通过各种参数告知内核需要什么，内核通过这些参数监听外部socket，一旦有事件发生，将事件通过`epoll`通道回调到链表上，待应用程序获取。

​	由上面可知，当事件就绪时，是**内核**通过回调机制放到缓冲区中，等待进程获取。进程通过 **epoll_wait()** 从缓冲区中获取已就绪的事件集合，而不是像`select`和`poll`通过轮询全部fd的方式获取就绪fd。

​	在`epoll`操作中，有 **epoll_event** 的结构类型，该结构类型用于保存事件类型和连接相关结构，表示某连接发生了什么事件。

------

### 工作模式

​	`epoll` 的描述符事件有两种触发模式：**LT（level trigger）**和 **ET（edge trigger）**。**LT模式是默认模式**。

​	**LT模式**与**ET模式**的区别：

- **LT模式**：当 **epoll_wait()** 检测到描述符事件发生并将此事件通知应用程序，**应用程序可以不立即处理该事件**。如果不处理，下次调用 **epoll_wait()** 时，还会再次通知应用程序处理该事件。
- **ET模式**：当 **epoll_wait()** 检测到描述符事件发生并将次事件通知应用程序，**应用程序必须立即处理该事件**。如果不处理，下次调用 **epoll_wait()** 时，不会再次通知应用程序处理该事件。

**LT 模式**

​	当 **epoll_wait()** 检测到描述符事件到达时，内核会将此事件通知进程，进程可以不立即处理该事件，下次调用 **epoll_wait()** 会再次通知进程。是默认的一种模式，并且同时支持 **Blocking** 和 **No-Blocking**。

**ET 模式**

​	和 LT 模式不同的是，通知之后进程必须立即处理事件，下次再调用 **epoll_wait()** 时不会再得到事件到达的通知。

​	很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。只支持 **No-Blocking**，**以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死**。

### 应用场景

​	很容易产生一种错觉认为只要用 `epoll` 就可以了，`select` 和 `poll` 都已经过时了，其实它们都有各自的使用场景。

#### select 应用场景

​	`select`的 **timeout** 参数精度为微秒，而 `poll` 和 `epoll` 为毫秒，因此 `select` 更加**适用于实时性要求比较高的场景，比如核反应堆的控制。**

​	`select` 可**移植性更好**，几乎被所有主流平台所支持。

#### poll 应用场景

​	`poll` 没有最大描述符数量的限制，如果平台支持并且对实时性要求不高，应该使用 `poll` 而不是 `select`。

#### epoll 应用场景

​	只需要运行在 Linux 平台上，有大量的描述符需要同时轮询，并且这些连接最好是长连接。

​	需要同时监控小于 1000 个描述符，就没有必要使用 `epoll`，因为这个应用场景下并不能体现 epoll 的优势。

​	需要监控的描述符状态变化多，而且都是非常短暂的，也没有必要使用 `epoll`。因为 `epoll`  中的所有描述符都存储在内核中，造成每次需要对描述符的状态改变都需要通过 **epoll_ctl()** 进行系统调用，频繁系统调用降低效率。并且  epoll 的描述符存储在内核，不容易调试。

# Reactor模式

​	Reactor模式是**一种能够处理一个或多个客户端并发交互服务请求的事件设计模式**。当请求到达后，服务器通过I/O多路复用策略，然后根据请求的服务派发给相应的处理程序。

## Reactor结构

![Reactor结构](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\Reactor结构.png)

​	上图列出了Reactor模式下，五种角色之间的关系，下面介绍这五种角色：

- **Handle**（句柄或描述符）：

    ​	**本质上表示一种资源，是由操作系统提供的**。该资源用于表示一个个的事件，事件既可以来自于外部，也可以来自于内部。外部事件比如说客户端的连接请求、客户端发送过来的数据等；内部事件比如说操作系统产生的定时事件等。就是相当于一个文件描述符，是个事件集合。

- **Synchronos Event Demultiplexer**（同步事件分离器）：

    ​	**本身是一个系统调用，用于等待事件发生**。调用方在调用它时会被阻塞，一直阻塞到同步事件分离器上有事件为止。对于Linux来说，同步事件分离器指的是常用的I/O多路复用机制，比如`select、poll、epoll`等。在Java NIO领域，同步分离器对应的组件就是**Selector**，对应的方法就是**select()**。

- **Event Handler**（事件处理器）：

    ​	**本身由多个回调方法构成，这些回调方法与构成了与应用相关的对于某个事件的反馈机制**（即对于已就绪的事件应做何种处理的回调机制）。在Java NIO领域中并没有提供事件处理器机制让我们调用或去进行回调，是由我们自己编写代码完成的。Netty 相比于Java  NIO来说，在事件处理器这个角色上进行了一个升级，它为我们开发者提供了大量的回调方法，**供我们在特定事件产生时实现相应的回调方法进行业务逻辑的处理**，即**ChannelHandler**。**ChannelHandler**中的方法对应的都是一个个事件的回调。

    ​	该角色是连接事件与具体事件处理器的桥梁，对于怎样的事件应该作何种事情，都是在该角色中定义。

- **Concrete Event Handler**（具体事件处理器）：

    ​	**是事件处理器的实现**。它本身实现了事件处理器所提供的各种回调方法，从而实现了特定于业务的逻辑。它本质上就是我们所编写的一个个的处理器实现。（就是实现做什么事情，或者说应用程序要做什么）

- **Initiation Dispatcher**（初始分发器）：

    ​	**实际上就是Reactor角色**。它本身定义了一些规范，这些规范用于控制事件的调度方式，同时又提供了应用进行事件处理器的注册、删除等设施。它本身是整个事件处理器的核心所在，**Initiation Dispatcher**会通过**Synchronous Event Demultiplexer**来等待事件的发生。一旦事件发生，**Initiation  Dispatcher**首先会分离出每一个事件，然后调用事件处理器，最后调用相关的回调方法来处理这些事件。Netty中**ChannelHandler**里的一个个回调方法都是由**bossGroup**或**workGroup**中的某个**EventLoop**来调用的。

    ​	这个角色相当于管理员的角色，事件处理器需要注册在该角色上才能够使服务器实现服务。



​	**个人总结**：Reactor就是一个处理并发连接请求的、基于事件的设计模式，主要是为了以并发的方式处理连接请求，所以也就采用了IO多路复用机制（事件IO）。Reactor分为了五种角色，这些角色的具体实现需要根据业务场景或技术支持作讨论，并不是与某种技术是一一对应的关系。

## Reactor模式流程

​	为简便，每个角色使用首字母大写缩写表示：

1. 初始化**ID**，然后将若干个**CEH**注册到**ID**中。当应用向**ID**注册**CEH**时，会在注册的同时指定感兴趣的事件，即应用会标识出该**EH**希望**ID**在某些事件发生时向其发出通知，事件通过**Handle**来标识，而**CEH**又持有该**Handle**。这样，事件 ——> **Handle** ——> **CEH**  就关联起来了。
2. **ID** 会要求每个**EH**向其传递内部的**Handle**。该**Handle**向操作系统标识了**EH**。（这里没看懂）
3. 当所有的**CEH**都注册完毕后，应用会调用**handle_events**方法来启动**ID**的事件循环。这时，**ID**会将每个注册的**CEH**的**Handle**合并起来，并使用**SED**(同步事件分离器)同步阻塞地等待事件的发生。比如说，TCP协议层会使用**select**同步事件分离器操作来等待客户端发送的数据到达连接的**socket handler**上。
4.  当与某个事件源对应的**Handle**变为**ready状态**时(比如说，TCP socket变为等待读状态时)，**SED**就会通知**ID**。
5. **ID**会触发事件处理器的回调方法，从而响应这个处于**ready状态**的**Handle**。当事件发生时，**ID**会将被事件源激活的**Handle**作为**key**来寻找并分发恰当的**EH**回调方法。
6. **ID**会回调**EH**的**handle_event(type)回调方法**来执行特定于应用的功能（开发者自己所编写的功能），从而响应这个事件。所发生的事件类型可以作为该方法参数并被该方法内部使用来执行额外的特定于服务的分离与分发。

## Reactor实现方式

### 单线程Reactor模式

![单线程Reactor模式](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\单线程Reactor模式.png)

​	以Java NIO说明流程如下：

1. 服务器端的Reactor是一个线程对象，该线程会**启动事件循环**，并使用 Selector 来实现IO的多路复用。**注册**一个 Acceptor事件处理器 到 Reactor 中，Acceptor事件处理器 所关注的事件是 ACCEPT事件，这样 Reactor 会**监听**客户端向服务器端发起的连接请求事件( ACCEPT事件 )。
2. 客户端向服务器端**发起一个连接请求**，Reactor 监听到了该 ACCEPT事件 的发生并将该 ACCEPT事件 **派发**给相应的 Acceptor处理器 来进行**处理**。Acceptor处理器 通过`accept()`方法得到与这个客户端对应的连接(SocketChannel)，然后将该连接所关注的 READ事件 以及对应的 READ事件处理器 **注册**到 Reactor 中，这样一来 Reactor 就会**监听**该连接的 READ事件 了。或者当你需要向客户端发送数据时，就向 Reactor 注册该连接的 WRITE事件和其处理器。
3. 当 Reactor **监听到有读或者写事件发生时**，将相关的事件派发给对应的处理器进行处理。比如，读处理器会通过 SocketChannel 的 `read()` 方法读取数据，此时 `read()` 操作可以直接读取到数据，而不会堵塞或等待可读的数据到来。
4. 每当处理完所有就绪的感兴趣的I/O事件后，Reactor线程 会**再次执行** `select()` 阻塞等待新的事件就绪并将其分派给对应处理器进行处理。

个人总结上述流程：先对客户端的连接请求进行处理，服务器端接受连接后再根据业务需求对某种连接注册相应的事件和事件处理器。也就是说一个连接对应的事件和事件处理器并不是固定的，是可变的。



​	注意，Reactor的单线程模式的单线程主要是针对于I/O操作而言，也就是所有的I/O的 `accept()、read()、write() 以及 connect()` 操作都在一个线程上完成的。

​	但在目前的单线程Reactor模式中，不仅I/O操作在该Reactor线程上，连非I/O的业务操作也在该线程上进行处理了，这可能会大大延迟I/O请求的响应。所以我们应该将非I/O的业务逻辑操作从Reactor线程上卸载，以此来加速Reactor线程对I/O请求的响应。

​	**个人总结**：就如[套接字NIO实例](#套接字NIO实例)中的代码就是一种单线程Reactor模式的实现，全部工作只有一个线程完成，包括IO操作和非IO操作。该模式有点像基于事件的BIO模式。

### 使用工作者线程池

![工作者线程池Reactor模式](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\工作者线程池Reactor模式.png)

​	**与单线程Reactor模式不同的是，添加了一个工作者线程池，并将非I/O操作（上图中的decode、compute、encode或为新连接注册其它时间和事件处理器）从Reactor线程中移出转交给工作者线程池来执行。**这样能够提高Reactor线程的I/O响应，不至于因为一些耗时的业务逻辑而延迟对后面I/O请求的处理。

使用线程池的优势：

- 通过重用现有的线程而不是创建新线程，可以在处理多个请求时分摊在线程创建和销毁过程产生的巨大开销。
- 另一个额外的好处是，当请求到达时，工作线程通常已经存在，因此不会由于等待创建线程而延迟任务的执行，从而提高了响应性。
- 通过适当调整线程池的大小，可以创建足够多的线程以便使处理器保持忙碌状态。同时还可以防止过多线程相互竞争资源而使应用程序耗尽内存或失败。



​	注意，在上图的改进的版本中，所以的I/O操作依旧由一个Reactor来完成，包括I/O的accept()、read()、write()以及connect()操作。

​	对于一些小容量应用场景，可以使用单线程模型。但是对于高负载、大并发或大数据量的应用场景却不合适，主要原因如下：

- 一个NIO线程同时处理成百上千的链路，性能上无法支撑，即便NIO线程的CPU负荷达到100%，也无法满足海量消息的读取和发送；
- 当NIO线程负载过重之后，处理速度将变慢，这会导致大量客户端连接超时，超时之后往往会进行重发，这更加重了NIO线程的负载，最终会导致大量消息积压和处理超时，成为系统的性能瓶颈；



### 多Reactor线程模式

![多Reactor线程模式](C:\Users\Administrator\Desktop\学习\java\Java-learn\img\多Reactor线程模式.png)

​	Reactor线程池中的每一Reactor线程都会有自己的Selector、线程和分发的事件循环逻辑。	

​	mainReactor可以只有一个，但subReactor一般会有多个。**mainReactor线程主要负责接收客户端的连接请求，然后将接收到的SocketChannel传递给subReactor，由subReactor来完成和客户端的通信**。

​	流程如下：

1. **注册**一个 Acceptor事件处理器 到 mainReactor中，Acceptor事件处理器 所关注的事件是 ACCEPT 事件，这样 mainReactor 会**监听**客户端向服务器端发起的连接请求事件(ACCEPT事件)。**启动** mainReactor 的事件循环。
2. 客户端向服务器端**发起一个连接请求**，mainReactor 监听到了该 ACCEPT事件 并将该 ACCEPT事件 **派发**给 Acceptor处理器 来进行**处理**。Acceptor处理器 通过 `accept()` 方法得到与这个客户端对应的连接(SocketChannel)，然后将这个 SocketChannel 传递给 subReactor 线程池。
3. subReactor 线程池分配一个 subReactor 线程给这个 SocketChannel，即将 SocketChannel 关注的 READ 事件以及对应的 READ事件处理器 注册到 subReactor 线程中。当然你也可注册 WRITE 事件以及 WRITE 事件处理器到 subReactor 线程中以完成I/O写操作。Reactor线程池 中的每一个 Reactor线程 都会有自己的 Selector、线程和分发的循环逻辑。
4. 当有**I/O事件就绪**时，相关的 subReactor 就将事件**派发** 给相应的处理器处理。注意，这里 subReactor 线程只负责完成I/O的`read()`操作，在读取到数据后将业务逻辑的处理放入到线程池（处理非IO操作的线程）中完成，若完成业务逻辑后需要返回数据给客户端，则相关的I/O的 `write` 操作还是会被提交回 subReactor 线程来完成。



​	注意，所以的I/O操作 (包括，I/O的accept()、read()、write()以及connect()操作) 依旧还是在Reactor线程(mainReactor线程 或 subReactor线程)中完成的。**Thread Pool(线程池)仅用来处理非I/O操作的逻辑**。



​	多Reactor线程模式将”**接受客户端连接请求**“和“**与客户端通信**”分在两个或多个Reactor线程完成。

​	**mainReactor线程负责接受客户端连接请求的操作**，不负责与客户端的通信，而是将建立好的连接转交给 subReactor线程；**subReactor线程主要负责与客户端的通信**。这样服务器端就不会因为IO操作数据量过大而导致连接请求不能即时处理的情况。并且subReactor线程可以通过线程池技术来与客户进行并发式的通信，在多核的操作系统中这能大大提升应用的负载和吞吐量。

### 总结

​	三种实现方式都与线程有关，有着单线程与多线程方式的区别，还有着应用多线程方式的不同。

​	要理解三者的区别，需要将应用中的IO操作和非IO操作分开，也需要理解Reactor的Accpt事件与其它事件，且Reactor模式的关注点是并发处理请求。

# 参考资料

[CyC2018之JavaIO]: https://github.com/CyC2018/CS-Notes/blob/master/notes/JavaIO.md

[同步、异步、阻塞、非阻塞IO总结（IO模型总结）]: https://blog.csdn.net/qq_36573828/article/details/89149057

[Linux IO模式及 select、poll、epoll详解]: https://segmentfault.com/a/1190000003063859

[Java NIO浅析]: https://tech.meituan.com/2016/11/04/nio.html

[Reactor模式详解]: https://www.cnblogs.com/winner-0715/p/8733787.html

