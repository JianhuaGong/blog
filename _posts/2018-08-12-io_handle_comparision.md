---
title: Java IO： BIO，NIO，AIO 
tags: netty, bio, nio
grammar_cjkRuby: true
---

# BIO
## 概念说明

**BIO（Blocking IO）**：同步阻塞IO，只要有请求过来，它的数据的读取或写入必须阻塞在一个线程上等待其完成。
Java1.4之前，都是BIO通信模式，而我们常见字节流和字符流也都是BIO。
![IO Stream Class](https://jianhuagong.github.io/blog/images/iostream.jpg)

BIO的通信机制：
1. 服务端启动一个ServerSocket并监听在某一端口上；
2. 然后由一个独立的Acceptor线程负责监听客户端的连接，这其中ServerSocket.accept()是阻塞的，直到有客户端请求才会返回，每个客户端连接后，服务端会启动一个线程去处理该客户端的请求；
3. 然后客户端启动Socket来对服务端进行连接；
4. 服务端Accepto接收到客户端连接请求之后，为每个客户端请求创建一个新的线程进行IO处理，通过输入流读取客户端发过来的数据（InputStream.read()方法时是阻塞的，它会一直等到数据到来时（或超时）才会返回），通过输出流返回应答给客户端，线程销毁等；
![enter description here](https://jianhuagong.github.io/blog/images/bio_mode.png)

## BIO服务端代码
``` javascript
public class BioEchoServer
{

    public static void main(String[] args)
    {
        BioEchoServer bioEchoServer = new BioEchoServer();
        try
        {
            bioEchoServer.serve(8080);
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }

    public void serve(int port) throws IOException
    {
        final ServerSocket socket = new ServerSocket(port); // 1
        System.out.println("Server listen to " + port);
        try
        {
            while (true)
            {
                final Socket clientSocket = socket.accept(); // 2
                System.out.println("Accepted connection from " + clientSocket);
                new Thread(new Runnable()
                {
                    public void run()
                    {
                        try
                        {
                            OutputStream out = clientSocket.getOutputStream();

                            byte[] bs = new byte[1024];
                            int len = 0;
                            while ((len = clientSocket.getInputStream().read(bs)) != -1)
                            {
                                String str = new String(bs, 0, len);
                                System.out.println("Server thread "
                                        + Thread.currentThread().getName() + " read: " + str);

                                out.write(str.getBytes());
                                out.flush();
                            }

                        }
                        catch (IOException e)
                        {
                            e.printStackTrace();
                            try
                            {
                                clientSocket.close();
                            }
                            catch (IOException ex)
                            {
                                // ignore on close
                            }
                        }
                    }
                }).start();
            }
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
        finally
        {
            socket.close();
        }
    }
}
```

## BIO客户端代码

``` javascript
public class BioClient
{

    public static void main(String[] args)
    {
        Socket socket = null;
        try
        {
            socket = new Socket("127.0.0.1", 8080);
            PrintWriter writer = new PrintWriter(socket.getOutputStream(), true);
            writer.write("hello, I'm BIO!");
            writer.flush();

            InputStream is = socket.getInputStream();
            byte[] bs = new byte[1024];
            int len = 0;
            while ((len = is.read(bs)) != -1)
            {
                System.out.println("Clinet thread " + Thread.currentThread().getName() + " read: "
                        + new String(bs, 0, len));

            }

        }
        catch (IOException e)
        {
            e.printStackTrace();
        }

        finally
        {
            try
            {
                socket.close();
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }

    }
}
```

# NIO
## 概念说明

**NIO（New IO 或 Non-blocking IO）**：同步非阻塞IO，一个有效请求（数据可读或可写）过来时才会分配线程去进行数据的读或写。
JDK 1.4中的 java.nio.* 包中引入新的Java New I/O库，它是基于事件驱动的思想来实现的，其目的就是解决BIO的大并发问题，在BIO模型中，如果需要并发处理多个I/O请求，需要多线程来支持。而NIO使用了多路复用器机制，以socket使用来说，多路复用器通过不断轮询各个连接的状态，只有在socket有流可读或者可写时，应用程序才需要去处理它，在线程的使用上，就不需要一个连接就必须使用一个处理线程了，而是只是有效请求时，才会使用一个线程去处理，这样就避免了BIO模型下大量线程处于阻塞等待状态的情景。NIO模型图如下：
![nio mode](https://jianhuagong.github.io/blog/images/nio_mode.png)

相对于BIO的流，NIO抽象出了新的通道（Channel）作为输入输出的通道，并且提供了缓存（Buffer）的支持，在进行读操作时，需要使用Buffer分配空间，然后将数据从Channel中读入Buffer中，对于Channel的写操作，也需要现将数据写入Buffer，然后将Buffer写入Channel中。

### 缓冲区 Buffer
Buffer是一个对象，包含一些要写入或者读出的数据。在NIO库中，所有数据都是用缓冲区来包装的，读取数据时，它是直接读到缓冲区中的；写入数据时，也是写入到缓冲区中。任何时候访问NIO中的数据，都是通过缓冲区进行操作。
    缓冲区实际上是一个数组，并提供了对数据结构化访问以及维护读写位置等信息。
    具体的缓存区有这些：ByteBuffe、CharBuffer、 ShortBuffer、IntBuffer、LongBuffer、FloatBuffer、DoubleBuffer，它们实现了接口：Buffer。

### 通道 Channel
我们对数据的读取和写入要通过Channel，它就像水管一样，是一个通道。通道不同于流的地方就是通道是双向的，可以用于读、写和同时读写操作。
底层的操作系统的通道一般都是全双工的，所以全双工的Channel比流能更好的映射底层操作系统的API。
Channel主要分两大类：
 - SelectableChannel：用于网络读写
 - FileChannel：用于文件操作
 
后面代码使用的ServerSocketChannel和SocketChannel都是SelectableChannel的子类。

### 多路复用器 Selector
Selector是Java  NIO 编程的基础。Selector提供选择已经就绪的任务的能力：Selector会不断轮询注册在其上的Channel，如果某个Channel上面发生读或者写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel的集合，进行后续的I/O操作。
一个Selector可以同时轮询多个Channel，因为JDK使用了epoll()代替传统的select实现，所以没有最大连接句柄1024/2048的限制。所以，只需要一个线程负责Selector的轮询，就可以接入成千上万的客户端。

## NIO服务端代码

``` javascript
import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class NioEchoServer
{
    public static void main(String[] args)
    {
        new Thread(() -> {
            NioEchoServer echoServer = new NioEchoServer();
            try
            {
                echoServer.serve(8080);
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }).start();
    }

    public void serve(int port) throws IOException
    {
        System.out.println("Listening for connections on port " + port);
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        ServerSocket serverSocket = serverChannel.socket();
        InetSocketAddress address = new InetSocketAddress(port);
        serverSocket.bind(address); // 1
        serverChannel.configureBlocking(false);
        Selector selector = Selector.open();
        serverChannel.register(selector, SelectionKey.OP_ACCEPT); // 2
        while (true)
        {
            try
            {
                selector.select(); // 3
            }
            catch (IOException ex)
            {
                ex.printStackTrace();
                break;
            }
            Set<SelectionKey> readyKeys = selector.selectedKeys(); // 4
            Iterator<SelectionKey> iterator = readyKeys.iterator();
            while (iterator.hasNext())
            {
                SelectionKey key = (SelectionKey) iterator.next();
                iterator.remove(); // 5
                try
                {
                    if (key.isAcceptable())
                    {
                        ServerSocketChannel server = (ServerSocketChannel) key.channel();
                        SocketChannel client = server.accept(); // 6
                        System.out.println("Accepted connection from " + client);
                        client.configureBlocking(false);
                        client.register(selector, SelectionKey.OP_WRITE | SelectionKey.OP_READ,
                                ByteBuffer.allocate(100)); // 7
                    }
                    else if (key.isReadable())
                    { // 8
                        SocketChannel client = (SocketChannel) key.channel();
                        ByteBuffer output = (ByteBuffer) key.attachment();
                        output.clear();
                        client.read(output); // 9
                        System.out.println("Server received: " + new String(output.array()));
                    }
                    else if (key.isWritable())
                    { // 10
                        SocketChannel client = (SocketChannel) key.channel();
                        ByteBuffer output = (ByteBuffer) key.attachment();

                        output.flip();
                        client.write(output); // 11
                        output.compact();
                    }
                }
                catch (IOException ex)
                {
                    key.cancel();
                    try
                    {
                        key.channel().close();
                    }
                    catch (IOException cex)
                    {}
                }
            }
        }
    }
}

```
## NIO客户端代码

``` javascript
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
import java.util.Set;

public class NioClient
{
    private static String DEFAULT_HOST = "127.0.0.1";
    private static int DEFAULT_PORT = 8080;
    private static Selector selector;
    private static SocketChannel socketChannel;

    public static void main(String[] args)
    {
        try
        {
            // 创建选择器
            selector = Selector.open();
            // 打开监听通道
            socketChannel = SocketChannel.open();
            // 如果为 true，则此通道将被置于阻塞模式；如果为 false，则此通道将被置于非阻塞模式
            socketChannel.configureBlocking(false);// 开启非阻塞模式
            socketChannel.connect(new InetSocketAddress(DEFAULT_HOST, DEFAULT_PORT));
            socketChannel.register(selector, SelectionKey.OP_CONNECT);
            while (true)
            {
                // 阻塞,只有当至少一个注册的事件发生的时候才会继续.
                selector.select();
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = keys.iterator();
                SelectionKey key = null;
                while (iterator.hasNext())
                {
                    key = iterator.next();
                    iterator.remove();
                    try
                    {
                        handleInput(key);
                    }
                    catch (Exception e)
                    {
                        if (key != null)
                        {
                            key.cancel();
                            if (key.channel() != null)
                            {
                                key.channel().close();
                            }
                        }
                    }
                }
            }
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }

    }

    private static void handleInput(SelectionKey key) throws IOException
    {
        if (key.isValid())
        {
            if (key.isConnectable())
            {
                SocketChannel channel = (SocketChannel) key.channel();
                if (channel.isConnectionPending())
                {
                    channel.finishConnect();
                }
                channel.configureBlocking(false);
                channel.register(selector, SelectionKey.OP_WRITE);
            }
            // 读消息
            if (key.isReadable())
            {
                // 创建ByteBuffer，并开辟一个1M的缓冲区
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                // 读取请求码流，返回读取到的字节数
                int readBytes = socketChannel.read(buffer);
                // 读取到字节，对字节进行编解码
                if (readBytes > 0)
                {
                    // 将缓冲区当前的limit设置为position=0，用于后续对缓冲区的读取操作
                    buffer.flip();
                    // 根据缓冲区可读字节数创建字节数组
                    byte[] bytes = new byte[buffer.remaining()];
                    // 将缓冲区可读字节数组复制到新建的数组中
                    buffer.get(bytes);
                    String result = new String(bytes, "UTF-8");
                    System.out.println("Client received: " + result);
                }
                // 没有读取到字节 忽略
                // else if(readBytes==0);
                // 链路已经关闭，释放资源
                else if (readBytes < 0)
                {
                    key.cancel();
                    socketChannel.close();
                }
                // socketChannel.register(selector, SelectionKey.OP_CONNECT);
            }
            else if (key.isWritable())
            {
                socketChannel.register(selector, SelectionKey.OP_READ);
                doWrite(socketChannel, "I'm NIO");
            }
        }
    }

    // 异步发送消息
    private static void doWrite(SocketChannel channel, String request) throws IOException
    {
        // 将消息编码为字节数组
        byte[] bytes = request.getBytes();
        // 根据数组容量创建ByteBuffer
        ByteBuffer writeBuffer = ByteBuffer.allocate(bytes.length);
        // 将字节数组复制到缓冲区
        writeBuffer.put(bytes);
        // flip操作
        writeBuffer.flip();
        // 发送缓冲区的字节数组
        channel.write(writeBuffer);
        // ****此处不含处理“写半包”的代码
    }
}

```

# AIO
## 概念说明
**AIO（Asynchronous IO）**：异步非阻塞IO，在此种模式下，用户进程只需要发起一个IO操作然后立即返回，等IO操作真正的完成以后，应用程序会得到IO操作完成的通知，此时用户进程只需要对数据进行处理就好了，不需要进行实际的IO读写操作，因为真正的IO读取或者写入操作已经由内核完成了。
从编程模式上来看AIO相对于NIO的区别在于，NIO需要使用者线程不停的轮询IO对象，来确定是否有数据准备好可以读了，而AIO则是在数据准备好之后，才会通知数据使用者，这样使用者就不需要不停地轮询了。当然AIO的异步特性并不是Java实现的伪异步，而是使用了系统底层API的支持，在Unix系统下，采用了epoll IO模型，而windows便是使用了IOCP模型。
在Jdk1.7中引入AIO，被称作NIO.2，主要在java.nio.channels包下增加了下面四个异步通道：
- AsynchronousSocketChannel
- AsynchronousServerSocketChannel
- AsynchronousFileChannel
- AsynchronousDatagramChannel

其中的read/write方法，会返回一个带回调函数的对象，当执行完读取/写入操作后，直接调用回调函数。

## AIO服务端代码

``` javascript
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.CountDownLatch;

public class Nio2EchoServer
{

    public static void main(String[] args)
    {
        Nio2EchoServer echoServer = new Nio2EchoServer();
        try
        {
            echoServer.serve(8080);
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }

    public void serve(int port) throws IOException
    {
        final AsynchronousServerSocketChannel serverChannel =
                AsynchronousServerSocketChannel.open();
        InetSocketAddress address = new InetSocketAddress(port);
        serverChannel.bind(address); // 1
        System.out.println("Listening for connections on port " + port);
        final CountDownLatch latch = new CountDownLatch(1);
        serverChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>()
        { // 2
            public void completed(final AsynchronousSocketChannel channel, Object attachment)
            {
                serverChannel.accept(null, this); // 3
                ByteBuffer buffer = ByteBuffer.allocate(100);
                channel.read(buffer, buffer, new EchoCompletionHandler(channel)); // 4
            }

            public void failed(Throwable throwable, Object attachment)
            {
                try
                {
                    serverChannel.close(); // 5
                }
                catch (IOException e)
                {
                    // ingnore on close
                }
                finally
                {
                    latch.countDown();
                }
            }
        });
        try
        {
            latch.await();
        }
        catch (InterruptedException e)
        {
            Thread.currentThread().interrupt();
        }
    }

    private final class EchoCompletionHandler implements CompletionHandler<Integer, ByteBuffer>
    {
        private final AsynchronousSocketChannel channel;

        EchoCompletionHandler(AsynchronousSocketChannel channel)
        {
            this.channel = channel;
        }

        public void completed(Integer result, ByteBuffer buffer)
        {
            System.out.println("Server received: " + new String(buffer.array()));

            buffer.flip();
            channel.write(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>()
            { // 6
                public void completed(Integer result, ByteBuffer buffer)
                {
                    if (buffer.hasRemaining())
                    {
                        channel.write(buffer, buffer, this); // 7
                    }
                    else
                    {
                        buffer.compact();
                        channel.read(buffer, buffer, EchoCompletionHandler.this); // 8
                    }
                }

                public void failed(Throwable exc, ByteBuffer attachment)
                {
                    try
                    {
                        channel.close();
                    }
                    catch (IOException e)
                    {
                        // ingnore on close
                    }
                }
            });
        }

        public void failed(Throwable exc, ByteBuffer attachment)
        {
            try
            {
                channel.close();
            }
            catch (IOException e)
            {
                // ingnore on close
            }
        }
    }
}
```

## AIO 客户端代码

``` javascript
import java.io.IOException;
import java.io.UnsupportedEncodingException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.util.concurrent.CountDownLatch;

public class Nio2Client implements CompletionHandler<Void, Nio2Client>
{
    private static Integer PORT = 8080;
    private static String IP_ADDRESS = "127.0.0.1";
    private static AsynchronousSocketChannel asynSocketChannel;

    private static CountDownLatch latch;

    public static void main(String[] args)
    {
        // 创建CountDownLatch等待
        latch = new CountDownLatch(1);
        try
        {
            asynSocketChannel = AsynchronousSocketChannel.open();
            // 打开通道
            asynSocketChannel.connect(new InetSocketAddress(IP_ADDRESS, PORT)); // 创建连接 和NIO一样

            String msg = "I'm AIO";
            byte[] req = msg.getBytes();
            ByteBuffer writeBuffer = ByteBuffer.allocate(req.length);
            writeBuffer.put(req);
            writeBuffer.flip();

            asynSocketChannel.write(writeBuffer, writeBuffer,
                    new WriteHandler(asynSocketChannel, latch));
            latch.await();
        }
        catch (Exception e)
        {
            e.printStackTrace();
            try
            {
                asynSocketChannel.close();
            }
            catch (IOException e1)
            {
                e1.printStackTrace();
            }

        }
    }

    @Override
    public void completed(Void result, Nio2Client attachment)
    {
        System.out.println("客户端成功连接到服务器...");

    }

    @Override
    public void failed(Throwable exc, Nio2Client attachment)
    {
        System.err.println("连接服务器失败...");
        exc.printStackTrace();
        try
        {
            asynSocketChannel.close();
            latch.countDown();
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }
}


class WriteHandler implements CompletionHandler<Integer, ByteBuffer>
{
    private AsynchronousSocketChannel clientChannel;
    private CountDownLatch latch;

    public WriteHandler(AsynchronousSocketChannel clientChannel, CountDownLatch latch)
    {
        this.clientChannel = clientChannel;
        this.latch = latch;
    }

    @Override
    public void completed(Integer result, ByteBuffer buffer)
    {
        // 完成全部数据的写入
        if (buffer.hasRemaining())
        {
            clientChannel.write(buffer, buffer, this);
        }
        else
        {
            // 读取数据
            ByteBuffer readBuffer = ByteBuffer.allocate(1024);
            clientChannel.read(readBuffer, readBuffer, new ReadHandler(clientChannel, latch));
        }
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment)
    {
        System.err.println("数据发送失败...");
        try
        {
            clientChannel.close();
            latch.countDown();
        }
        catch (IOException e)
        {}
    }
}


class ReadHandler implements CompletionHandler<Integer, ByteBuffer>
{
    private AsynchronousSocketChannel clientChannel;
    private CountDownLatch latch;

    public ReadHandler(AsynchronousSocketChannel clientChannel, CountDownLatch latch)
    {
        this.clientChannel = clientChannel;
        this.latch = latch;
    }

    @Override
    public void completed(Integer result, ByteBuffer buffer)
    {
        buffer.flip();
        byte[] bytes = new byte[buffer.remaining()];
        buffer.get(bytes);
        String body;
        try
        {
            body = new String(bytes, "UTF-8");
            System.out.println("Client received: " + body);
        }
        catch (UnsupportedEncodingException e)
        {
            e.printStackTrace();
        }
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment)
    {
        System.err.println("数据读取失败...");
        try
        {
            clientChannel.close();
            latch.countDown();
        }
        catch (IOException e)
        {}
    }
}
```

# 小结
BIO，同步阻塞式IO，简单理解：一个连接一个线程，线程等待整个读或写处理完成，基于顺序模式；
NIO，同步非阻塞IO，简单理解：一个IO请求一个线程，线程通过轮询方式得到Ready事件后，继续处理读或写逻辑，基于Reactor模式；
AIO，异步非阻塞IO，简单理解：一个IO请求不需要用户线程，把IO的工作分出去由操作系统处理，仅读和写完成事件后回调注册handler。
![](https://jianhuagong.github.io/blog/images/io_mode.png)
