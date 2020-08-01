# Netty 实现服务端和客户端


在Netty学习笔记(一) 实现DISCARD服务中，我们使用Netty和Python实现了简单的丢弃DISCARD服务，这篇，我们使用Netty实现服务端和客户端交互的需求。
<a name="f076b6d8"></a>
## 前置工作

<a name="29c80db5"></a>
### 开发环境

- JDK8
- Netty版本:5.0.0.Alpha2
- 集成环境:IDEA
- 构建工具：Gradle

<a name="6860b943"></a>
### 依赖

```gradle
    compile group: 'io.netty', name: 'netty-all', version: '5.0.0.Alpha2'
    compile group: 'org.projectlombok', name: 'lombok', version: '1.18.0'
```

<a name="55abea2d"></a>
## 服务端

Netty服务器主要由两部分组成:

- 配置服务器功能，如线程、端口
- 实现服务器处理程序

<a name="333e18a1"></a>
### 服务端HandleAdapter

我们是首先实现服务端处理程序，实现Handle要求继承HandleAdapter,这里我们继承`SimpleChannelInboundHandler<T>`,下面是具体实现的代码信息,其每个函数的作用，我们只需要重写我们所需要的方法即可。

```java

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

/**
 * Netty 服务Handle
 *
 * @author tao
 */
@ChannelHandler.Sharable
public class EchoServiceHandle extends SimpleChannelInboundHandler<String> {

  /**
   * 接收到新的消息
   * @param ctx
   * @param msg
   * @throws Exception
   */
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    // 打印接收到的消息
    System.out.println("Netty服务端接收到消息 " + msg);
    // 回复消息
    ctx.channel().writeAndFlush("Send ----> 客户端" + ctx.channel().id() + "你好，我已经接收到你发送的消息");
  }

  /**
   * 有新的连接加入
   * @param ctx
   * @throws Exception
   */
  @Override
  public void channelActive(ChannelHandlerContext ctx) throws Exception {
    System.out.println("接入新的Channel,id = " + ctx.channel().id());
  }

  /**
   * 服务端发生异常信息的时候
   * @param ctx
   * @param cause
   */
  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    // 服务端发生异常
    System.out.println("Netty 服务端发生异常 ,异常信息：" + cause);
    ctx.close();
  }

  @Override
  protected void messageReceived(ChannelHandlerContext ctx, String msg) {}
}
```

<a name="73e3272d"></a>
### 服务端启动

在服务端的EchoServiceHandle完成之后，我们需要配置服务器的参数信息，比如端口等

```java

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import lombok.Data;

/**
 * @author tao
 */
@Data
public class EchoService {

  private int port;

  public void start() throws Exception {
    EventLoopGroup boosGroup = new NioEventLoopGroup();
    EventLoopGroup workGroup = new NioEventLoopGroup();
    try {
      ServerBootstrap bootstrap = new ServerBootstrap();
      bootstrap
          .group(boosGroup, workGroup)
          .channel(NioServerSocketChannel.class)
          .option(ChannelOption.SO_BACKLOG, 128)
          .childHandler(
              new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                  ChannelPipeline pipeline = ch.pipeline();
                  //字符串解码器
                  pipeline.addLast(new StringDecoder());
                  //字符串编码器
                  pipeline.addLast(new StringEncoder());
                  //服务端处理器
                  pipeline.addLast(new EchoServiceHandle());
                }
              });
      ChannelFuture sync = bootstrap.bind(port).sync();
      System.out.println("Netty Service start with " + port + "...");
      sync.channel().closeFuture().sync();
    } finally {
      workGroup.shutdownGracefully();
      boosGroup.shutdownGracefully();
    }
  }

  public static void main(String[] args) throws Exception {
    EchoService service = new EchoService();
    service.setPort(8080);
    service.start();
  }
}
```

启动后，我们可以看到启动信息如下:

```
Netty Service start with 8080...
```

<a name="efc6882b"></a>
## 客户端

客户端和服务端类似,可以相互参考学习.

<a name="27407e54"></a>
### 客户端HandleAdapter

```java
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.util.CharsetUtil;

@ChannelHandler.Sharable
public class EchoClientHandle extends SimpleChannelInboundHandler<String> {

  /**
   * 连接创建成功的时候
   * @param ctx
   * @throws Exception
   */
  @Override
  public void channelActive(ChannelHandlerContext ctx) throws Exception {
    System.out.println("channelActive");
	//连接成功后向服务端发送问候消息
    ctx.channel().writeAndFlush(Unpooled.copiedBuffer("你好，这里是Netty客户端", CharsetUtil.UTF_8));
  }

  /**
   * 发生异常
   * @param ctx
   * @param cause
   */
  @Override
  public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    System.out.println("exceptionCaught");
    cause.printStackTrace();
    ctx.close();
  }

  /**
   * 接收到服务端发过来的数据
   * @param ctx
   * @param msg
   */
  @Override
  public void channelRead(ChannelHandlerContext ctx, Object msg) {
   //打印接收到的数据信息
    System.out.println("channelRead = " + msg);
  }

  @Override
  protected void messageReceived(ChannelHandlerContext ctx, String msg) {}
}
```

<a name="43011f45"></a>
### 客户端启动

```java
import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import lombok.Data;
import lombok.experimental.Accessors;
import lombok.extern.slf4j.Slf4j;

import java.net.InetSocketAddress;

/** @author tao */
@Data
@Accessors(chain = true)
@Slf4j
public class EchoClient {

  private int port;

  private String host;

  public void start() throws Exception {
    EventLoopGroup group = new NioEventLoopGroup();
    try {
      Bootstrap bootstrap = new Bootstrap();
      bootstrap
          .group(group)
          .channel(NioSocketChannel.class)
          .handler(
              new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel ch) throws Exception {
                  ChannelPipeline pipeline = ch.pipeline();
                  // 字符串解码器
                  pipeline.addLast(new StringDecoder());
                  // 字符串编码器
                  pipeline.addLast(new StringEncoder());

                  pipeline.addLast(new EchoClientHandle());
                }
              });
      ChannelFuture sync = bootstrap.connect(new InetSocketAddress(host, port)).sync();
      sync.channel().closeFuture().sync();

    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      group.shutdownGracefully().sync();
    }
  }

  public static void main(String[] args) throws Exception {
    EchoClient client = new EchoClient();
    client.setHost("127.0.0.1").setPort(8080);
    client.start();
  }
}
```

<a name="7e8b2fa2"></a>
## 效果

<a name="48205952"></a>
### 服务端效果

```
   Netty Service start with 8080...
   接入新的Channel,id = c0d4f037
   Netty服务端接收到消息 你好，这里是Netty客户端
```

<a name="1c43964e"></a>
### 客户端效果

```
channelActive
channelRead = Send ----> 客户端c0d4f037你好，我已经接收到你发送的消息
```