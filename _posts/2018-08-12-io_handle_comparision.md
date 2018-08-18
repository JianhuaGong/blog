---
title: Java IO： BIO，NIO，AIO 
tags: netty, bio, nio
grammar_cjkRuby: true
---

**BIO（Blocking IO）**：同步阻塞IO，一个请求的数据的读取或写入必须阻塞在一个线程上等待其完成。Java1.4之前，都是BIO通信模式，而我们常见字节流和字符流也都是BIO。
![enter description here](./images/iostream.jpg | prepend: site.baseurl)

BIO的通信机制：
1. 服务端r启动一个ServerSocket并监听在某一端口上；
2. 然后由一个独立的Acceptor线程负责监听客户端的连接，这其中ServerSocket.accept()是阻塞的，直到有客户端请求才会返回，每个客户端连接后，服务端会启动一个线程去处理该客户端的请求；
3. 然后客户端启动Socket来对服务端进行连接；
4. 服务端Accepto接收到客户端连接请求之后，为每个客户端请求创建一个新的线程进行IO处理，通过输入流读取客户端发过来的数据（InputStream.read()方法时是阻塞的，它会一直等到数据到来时（或超时）才会返回），通过输出流返回应答给客户端，线程销毁等；
![enter description here](./images/bio_mode.png)

BIO服务端代码：
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
BIO客户端代码：

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