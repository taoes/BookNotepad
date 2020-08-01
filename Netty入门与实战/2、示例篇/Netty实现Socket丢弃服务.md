# 使用Netty实现Socket丢弃服务


官方那个给出的介绍是：Netty是由JBOSS提供的一个java开源框架。Netty提供异步的、事件驱动的网络应用程序框架和工具，用以快速开发高性能、高可靠性的网络服务器和客户端程序。然后我们简单理解一下，这玩意就是个程序，干什么的？netty是封装java socket noi的。 类似的功能是 apache的mina。
<a name="220e2c38"></a>
# 使用Netty实现Socket丢弃服务

相对于Tomcat这种Web Server（顾名思义主要是提供Web协议相关的服务的），Netty是一个Network Server，是处于Web Server更下层的网络框架，也就是说你可以使用Netty模仿Tomcat做一个提供HTTP服务的Web容器。`其实一个好使的处理Socket的东西`

<a name="2932f104"></a>
## 实现丢弃服务

这里插一下，就是我们的的通信是建立在一定的协议之上的，就比如我们常用的Web工程，前台（浏览器）发送一个请求，后台做出相应返回相应的结果，这个SOCKET通信的过程亦是如此。

在netty官方指南里面有讲，世上最简单的协议不是'Hello, World!' 而是 DISCARD(抛弃服务)。这个协议将会抛弃任何收到的数据，而不响应。就是你客户端发送消息，好，发送过去了，服务器也收到了，但是抛弃了。

说白了，就是你发一条消息给我，我收到了，仅此而已，不做任何响应。下面我们看看大致的步骤。

- 创建项目,添加Netty 依赖
- 实现丢弃服务
- 运行服务
- 使用Python进行测试

<a name="66a8e374"></a>
### 创建项目,添加Netty 依赖

使用IDEA创建一个普通项目，然后添加jar包或者直接创建一个maven项目也行。

在Maven中搜索netty-all 即可 地址是 [http://mvnrepository.com/artifact/io.netty/netty-all](http://mvnrepository.com/artifact/io.netty/netty-all) 建议选择5.0.0以上的版本。<br />或者直接创建Maven项目，引入依赖如下

```xml

<!-- https://mvnrepository.com/artifact/io.netty/netty-all -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>5.0.0.Alpha2</version>
</dependency>
```

<a name="2932f104-1"></a>
### 实现丢弃服务

```java
package com.company;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerAdapter;
import io.netty.channel.ChannelHandlerContext;
import io.netty.util.CharsetUtil;
import io.netty.util.ReferenceCountUtil;

/**
 * 本文件由周涛创建,位于com.company包下
 * 创建时间2018/4/20 11:46
 * 邮箱:zhoutao@xiaodouwangluo.com
 * 作用:实现丢弃服务
 *
 * @author tao
 */
public class DiscardServerHandle extends ChannelHandlerAdapter {

    /**
     * 接收到SOCKET的时候会调用此方法
     * @param ctx
     * @param msg
     * @throws Exception
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        try {
			//获取到接收的内容，并且实现
            ByteBuf in = (ByteBuf) msg;
            String message = in.toString(CharsetUtil.UTF_8);
            System.out.println(message);
        } finally {
            ReferenceCountUtil.release(msg);
        }
    }


    /**
     * 有新的连接加入的时候
     * @param ctx
     * @throws Exception
     */
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        System.out.println("新增Channel ,ChannelId = " + ctx.channel().id());
    }

    /**
     * 有连接断开被移除的时候调用
     * @param ctx
     * @throws Exception
     */
    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        System.out.println("移除Channel ,ChannelId = " + ctx.channel().id());
    }

    /**
     * 发生异常的时候调用
     * @param ctx
     * @param cause
     * @throws Exception
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();

    }
}
```

<a name="f13da9a6"></a>
### 运行服务

```java

package com.company;


import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;

/**
 * 本文件由周涛创建,位于com.company包下
 * 创建时间2018/4/20 11:46
 * 邮箱:zhoutao@xiaodouwangluo.com
 * 作用:创建运行服务
 *
 * @author tao
 */
public class DiscardServer {

    private int port;

    public DiscardServer(int port) {
        this.port = port;
    }

    public void run() throws InterruptedException {
        EventLoopGroup boos = new NioEventLoopGroup();
        EventLoopGroup worker = new NioEventLoopGroup();
        System.out.println("准备运行在端口:" + String.valueOf(this.port));

        try {

            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(boos, worker)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            socketChannel.pipeline().addLast(new DiscardServerHandle());
                        }
                    });

            ChannelFuture sync = serverBootstrap.bind(port).sync();
            sync.channel().closeFuture().sync();

        } finally {
            worker.shutdownGracefully();
            boos.shutdownGracefully();
        }

    }

    public static void main(String[] args) {
        DiscardServer server = new DiscardServer(8080);
        try {
            server.run();
        } catch (InterruptedException e) {
            System.out.println("发生了异常信息,异常信息如下所示：");
            e.printStackTrace();
        }
    }
}
```

<a name="09f09c1d"></a>
### 使用Python进行测试

python内置了socket库可以实现连接，我们这里使用Pyhton3.x来进行操作socket.当然这里只是演示SOCKET,你也可以使用其他方法尝试连接socket比如 js,java，或者talnet命令。

```python

#!/usr/bin/env python
# encoding: utf-8

#coding=utf-8
__author__ = '药师Aric'
'''
client端
长连接，短连接，心跳
'''
import socket
import time
host = '127.0.0.1'
port = 8080
client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client.setsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE, 1) #在客户端开启心跳维护
client.connect((host, port))
send_count = 0
try:
    while True:
        timeStr= time.strftime("%Y-%m-%d %H:%M:%S", time.localtime())
        client.send((u"你好，这是Python客户端,已经您发送消息 %s" % timeStr).encode())
        send_count = send_count + 1
        print("发送数据完成,已发送次数：%d" % send_count)
        time.sleep(2)
except:
    print("当前系统已经和服务器断开连接....")
```

运行Pyhton 代码，尝试连接SOCKET，执行一会之后停止pyhton ，结果如下:

<a name="41f2afd6"></a>
#### python发送的数据

```
/Users/tao/Code/Python/redisDemo/venv/bin/python /Users/tao/Code/Python/redisDemo/main/index.py
发送数据完成,已发送次数：1
发送数据完成,已发送次数：2
发送数据完成,已发送次数：3
发送数据完成,已发送次数：4
发送数据完成,已发送次数：5
当前系统已经和服务器断开连接....
```

<a name="576f862e"></a>
#### Netty收到的数据

![](https://cdn.nlark.com/yuque/0/2020/png/437981/1584415921105-0c6da7c0-5ced-46b2-8607-df631691c4de.png#align=left&display=inline&height=239&originHeight=239&originWidth=590&size=0&status=done&style=none&width=590)