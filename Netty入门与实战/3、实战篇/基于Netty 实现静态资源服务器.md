# 基于Netty 实现WebService 框架


<br />昨天在继续完善基于Netty构建的聊天室系统的过程中，发现了一个有意思的知识点，特此拿来做一个简单的静态网页服务器，好好的玩一玩Netty。<br />
<br />但是不管怎么说利用netty实现各种功能的流程都是类似的<br />

- 配置ServerHandle
- (可选)实现自定义的编码器
- 完成ServerBootStarp的配置
- 启动服务
- 连接到该服务


<br />好的，那么我们基于此来实现一个简单静态网页需求，要求实现能够通过地址访问html,js,css，以及图片等资源文件，那么开始吧<br />

<a name="f2902847"></a>
## 1、静态网页资源服务器


<a name="HttpServerHandleAdapter"></a>
### 1.1 Handler实现

<br />这里是最为复杂的步骤，具体代码可以看注释。<br />

```java
package com.zhoutao123.simpleChat.html;

import io.netty.channel.*;
import io.netty.handler.codec.http.*;
import io.netty.handler.ssl.SslHandler;
import io.netty.handler.stream.ChunkedNioFile;

import java.io.File;
import java.io.RandomAccessFile;

public class HttpServerHandleAdapter extends SimpleChannelInboundHandler<FullHttpRequest> {

    // 资源所在路径
    private static final String location;

    // 404文件页面地址
    private static final File NOT_FOUND;

    static {
        // 构建资源所在路径，此处参数可优化为使用配置文件传入
        location = "/home/tao/code/resource";
        // 构建404页面
        String path = location + "/404.html";
        NOT_FOUND = new File(path);
    }


    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {
        // 获取URI
        String uri = request.getUri();
        // 设置不支持favicon.ico文件
        if ("favicon.ico".equals(uri)) {
            return;
        }
        // 根据路径地址构建文件
        String path = location + uri;
        File html = new File(path);

        // 状态为1xx的话，继续请求
        if (HttpHeaders.is100ContinueExpected(request)) {
            send100Continue(ctx);
        }

        // 当文件不存在的时候，将资源指向NOT_FOUND
        if (!html.exists()) {
            html = NOT_FOUND;
        }

        RandomAccessFile file = new RandomAccessFile(html, "r");
        HttpResponse response = new DefaultHttpResponse(request.getProtocolVersion(), HttpResponseStatus.OK);

        // 文件没有发现设置状态为404
        if (html == NOT_FOUND) {
            response.setStatus(HttpResponseStatus.NOT_FOUND);
        }

        // 设置文件格式内容
        if (path.endsWith(".html")){
            response.headers().set(HttpHeaders.Names.CONTENT_TYPE, "text/html; charset=UTF-8");
        }else if(path.endsWith(".js")){
            response.headers().set(HttpHeaders.Names.CONTENT_TYPE, "application/x-javascript");
        }else if(path.endsWith(".css")){
            response.headers().set(HttpHeaders.Names.CONTENT_TYPE, "text/css; charset=UTF-8");
        }

        boolean keepAlive = HttpHeaders.isKeepAlive(request);

        if (keepAlive) {
            response.headers().set(HttpHeaders.Names.CONTENT_LENGTH, file.length());
            response.headers().set(HttpHeaders.Names.CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
        }
        ctx.write(response);

        if (ctx.pipeline().get(SslHandler.class) == null) {   
            ctx.write(new DefaultFileRegion(file.getChannel(), 0, file.length()));
        } else {
            ctx.write(new ChunkedNioFile(file.getChannel()));
        }
        // 写入文件尾部
        ChannelFuture future = ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);  
        if (!keepAlive) {
            future.addListener(ChannelFutureListener.CLOSE);
        }
        file.close();
    }

    private static void send100Continue(ChannelHandlerContext ctx) {
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE);
        ctx.writeAndFlush(response);
    }
}
```


<a name="HttpServerInitializer"></a>
### 1.2 HttpServerInitializer

<br />添加我们刚刚完成的HttpServerHandleAdapter<br />

```java
package com.zhoutao123.simpleChat.html;


import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.socket.SocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.stream.ChunkedWriteHandler;


public class HttpServerInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        //将请求和应答消息编码或解码为HTTP消息
        pipeline.addLast(new HttpServerCodec());
        //将HTTP消息的多个部分组合成一条完整的HTTP消息
        pipeline.addLast(new HttpObjectAggregator(64 * 1024));
        pipeline.addLast(new ChunkedWriteHandler());
        pipeline.addLast(new HttpServerHandleAdapter());
    }
}
```


<a name="b19ba4ef"></a>
### 1.3 服务启动


```java
public class Server {

    private final static int  port = 8080;
    
    public static void main(String[] args) throws InterruptedException {
        NioEventLoopGroup boss = new NioEventLoopGroup();
        NioEventLoopGroup work = new NioEventLoopGroup();

        try {
            ServerBootstrap serverBootstrap = new ServerBootstrap();
            serverBootstrap.group(boss, work)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new HttpServerInitializer())
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);
            ChannelFuture future = serverBootstrap.bind(port).sync();
            // 等待服务器  socket 关闭 。
            // 在这个例子中，这不会发生，但你可以优雅地关闭你的服务器。
            future.channel().closeFuture().sync();
        } finally {
            work.shutdownGracefully();
            boss.shutdownGracefully();
        }
    }
}
```


<a name="35990663"></a>
## 2、静态资源测试文件

<br />静态资源文件这里主要测试html以及js和css文件，这里写了3个html文件(404.html用于当输入路径不存在的时候，跳转到的文件)，一个js文件，一个css文件，一个logo图片，测试js和css在index.html文件中测试<br />
<br />需要注意的是这些资源文件需要放在上面代码中的location指定的位置，否则可能会出现访问不到的异常情况<br />

- index.html 测试文件



```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Netty静态资源服务器</title>
    <link rel="stylesheet" href="css/style.css"/>

</head>
<body>
    <h1 id="title">你好，欢迎来到基于Netty构建的静态资源服务器主页</h1>
    <h1 style="color: blue">当前位置Index.html</h1>
    <input type="button" value="启动问候语" onclick="sayHello()"/>
    <a href="about.html">关于Netty和作者</a>
</body>
    <script src="js/message.js"></script>
</html>
```


- about 测试文件



```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>WebSocket Chat</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
<h1 style="color: blue">这是关于界面</h1>
<img style="width: 250px" src="logo.jpg">
<a href="index.html">点击我，返回主页</a>
</body>
</html>
```


- 404 测试文件



```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>WebSocket Chat</title>
</head>
<body>
<h1 style="color: red">没有发现此页面</h1>
</body>
</html>
```


- js 测试文件



```javascript
function sayHello() {
    alert("你好,欢迎使用Netty服务器")
}
```


- css 测试文件



```css
#title{
    color: red;
    text-underline: darkcyan;
}

input{
    background-color: darkcyan;

}
```

<br />

```
.
├── index.html
├── 404.html
├── about.html
├── logo.jpg
├── css
│   └── style.css
└── js
    └── message.js

2 directories, 5 files
```


<a name="8ab3a61c"></a>
## 3、测试效果

<br />浏览器输入URL即可访问主页，可以看到访问成功，地址是localhost:8080/index.html ,开发者工具中展示状态均为200<br />
<br />![](https://cdn.nlark.com/yuque/0/2020/png/437981/1596268825334-7bd95a33-60df-424f-8571-5bd969bbdd35.png#align=left&display=inline&height=399&margin=%5Bobject%20Object%5D&originHeight=399&originWidth=1320&size=0&status=done&style=none&width=1320)<br />访问js点击启动问候语会执行js脚本弹出窗口<br />
<br />![](https://cdn.nlark.com/yuque/0/2020/png/437981/1596268825234-e71186a8-1e72-4509-aa92-53b14f5114b9.png#align=left&display=inline&height=246&margin=%5Bobject%20Object%5D&originHeight=246&originWidth=963&size=0&status=done&style=none&width=963)<br />
<br />
<br />点击超链接 ，即可跳转到about.html界面，这里引用了一个jpg图片 ，当输入一个不存在的文件的时候，可以看到状态为404.<br />
<br />![](https://cdn.nlark.com/yuque/0/2020/png/437981/1596268825154-0658d72e-d46c-4632-9122-dd488c58fa46.png#align=left&display=inline&height=376&margin=%5Bobject%20Object%5D&originHeight=376&originWidth=1012&size=0&status=done&style=none&width=1012)<br />

<a name="25f9c7fa"></a>
## 4、总结

<br />基于Netty实现的一个简单的静态资源服务器，可以说实现了基本的功能，但是还有其他很多idea可以实现，如负载均衡，设置配置文件(如资源所在路径以及端口等信息)，