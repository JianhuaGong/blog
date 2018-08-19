---
title: 第一个Netty程序 
tags: netty
grammar_cjkRuby: true
---

# 服务端程序

``` javascript
import java.net.InetSocketAddress;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelHandler.Sharable;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.util.CharsetUtil;

public class EchoServer
{
    private final int port;

    public EchoServer(int port)
    {
        this.port = port;
    }

    public static void main(String[] args) throws Exception
    {

        int port = Integer.parseInt("8080"); // 1
        new EchoServer(port).start(); // 2
    }

    public void start() throws Exception
    {
        NioEventLoopGroup group = new NioEventLoopGroup(); // 3
        try
        {
            ServerBootstrap b = new ServerBootstrap();
            b.group(group) // 4
                    .channel(NioServerSocketChannel.class) // 5
                    .localAddress(new InetSocketAddress(port)) // 6
                    .childHandler(new ChannelInitializer<SocketChannel>()
                    { // 7
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception
                        {
                            ch.pipeline().addFirst(new EchoServerHandler())
                                    .addLast(new EchoServerHandler());
                        }
                    });

            ChannelFuture f = b.bind().sync(); // 8
            System.out.println(EchoServer.class.getName() + " started and listen on "
                    + f.channel().localAddress());
            f.channel().closeFuture().sync(); // 9
        }
        finally
        {
            group.shutdownGracefully().sync(); // 10
        }
    }
}


@Sharable
class EchoServerHandler extends ChannelInboundHandlerAdapter
{

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception
    {
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("Server received: " + buf.toString(CharsetUtil.UTF_8));
        ctx.write(buf);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception
    {
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER).addListener(ChannelFutureListener.CLOSE);

    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception
    {
        cause.printStackTrace();
        ctx.close();
    }

}

```

# 客户端程序

``` javascript
import java.net.InetSocketAddress;

import io.netty.bootstrap.Bootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.ChannelHandler.Sharable;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.util.CharsetUtil;

public class EchoClient
{
    private final String host;
    private final int port;

    public EchoClient(String host, int port)
    {
        this.host = host;
        this.port = port;
    }

    public void start() throws Exception
    {
        EventLoopGroup group = new NioEventLoopGroup();
        try
        {
            Bootstrap b = new Bootstrap(); // 1
            b.group(group) // 2
                    .channel(NioSocketChannel.class) // 3
                    .remoteAddress(new InetSocketAddress(host, port)) // 4
                    .handler(new ChannelInitializer<SocketChannel>()
                    { // 5
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception
                        {
                            ch.pipeline().addLast(new EchoClientHandler());
                        }
                    });

            ChannelFuture f = b.connect().sync(); // 6

            f.channel().closeFuture().sync(); // 7
        }
        finally
        {
            group.shutdownGracefully().sync(); // 8
        }
    }

    public static void main(String[] args) throws Exception
    {

        final String host = "127.0.0.1";
        final int port = 8080;

        new EchoClient(host, port).start();
    }
}


@Sharable
class EchoClientHandler extends SimpleChannelInboundHandler<ByteBuf>
{

    @Override
    public void channelActive(ChannelHandlerContext ctx)
    {
        ctx.writeAndFlush(Unpooled.copiedBuffer("Netty rocks!", // 2
                CharsetUtil.UTF_8));
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception
    {
        System.out.println("Client received: " + msg.toString(CharsetUtil.UTF_8)); // 3

    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
    { // 4
        cause.printStackTrace();
        ctx.close();
    }

}
```