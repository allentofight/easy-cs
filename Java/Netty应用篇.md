@[toc]
## 前言

本文是 Netty 三讲系列文章的最后一讲：Netty 应用篇。在前两篇文章中，第一篇介绍了 Netty 的架构（[点我查看第一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)），第二篇对 Netty 关键源码进行了解析（[点我查看第二篇文章](https://mp.weixin.qq.com/s/F3uUqgEMxX3-sQPFHVFGow)），希望对大家理解 Netty 的工作原理有所帮助。Netty 的设计初衷就是为使用者提供更好的网络编程基础设施，因此在了解了 Netty 工作原理之后，本文向大家介绍使用 Netty 进行网络 IO 程序开发的一些关键问题，并给出一些 Demo。

## 1. 使用 Netty 的基本模式

下面这张图在前两篇文章中均有讲解，本文再次贴出来。它明确显示了使用者基于 Netty 开发自己的网络 IO 程序的工作重点，那就是：定义自己的 Handler，放入 Pipeline，以处理 IO 事件。


![](https://img-blog.csdnimg.cn/img_convert/8f157ac667a0d19d61191f7784b65ef1.png)


这些 Handler 可以是编码器 Handler、可以是解码器 Handler，也可以是业务处理 Handler。并且多个 Handler 组成一个责任链，IO 事件会流经整个责任链。若 IO 事件是出站事件，则在责任链中的每个 OutboundHandler 中得到处理，若 IO 事件是入站事件，则在责任链中的每个 InboundHandler 中得到处理。

而连接的建立、释放、Channel 的维护、Selector 的维护、IO 事件的捕捉等等细节均托管给 Netty。因此本文也把讲解的重点放在 Handler 上。

## 2. Tcp 粘包拆包处理

### 2.1. Tcp 粘包拆包的概念

TCP 是面向连接的，面向字节流的，提供可靠性传输服务。收发两端维护了一对 socket，发送端为了将多个 TCP 数据包更有效地传输给接收端，使用了 Nagle 算法进行了优化。Nagle 算法的基本思想是：将多个发送时间间隔较小、且数据包尺寸较小的 TCP 数据包合并成一个较大的 TCP 数据包进行传输。

这种批量数据处理方法虽然提高了传输效率，但也带来了一定的问题：因为面向流的通信是无消息边界保护的，因此接收端难以分辨出完整的数据包（读者如果暂时无法理解这句话，暂且留个印象，下文会用实例解释）。由于 TCP 无消息保护边界，就需要接收端处理消息边界问题。

以上两段就是所谓的 TCP 粘包、拆包问题。

下面给出粘包和拆包的图示。假如 Client 要通过 TCP 协议分别向 Server 发送两个数据包，两个数据包分别包含两段数据 data1 和 data2（比如两个序列化之后的 bean 对象），在 Nagle 算法的作用下，可能存在如下几种情形：

1）Server 分两次读到了两个独立的数据包，分别是 pkg1 和 pkg2，各自包含完整的 data1 和 data2，没有发生粘包和拆包；

2）Server 读到了一个 pkg1，data1 和 data2 被粘在一起放入其中。也就是说 Client 上发生了粘包，Server 收到 pkg1 后就要进行拆包处理；

3）Server 分两次读到了两个独立的数据包，分别是 pkg1 和 pkg2，其中 pkg1 包含了完整的 data1 和一部分 data2、pkg2 包含了另一部分 data2。也就是说 Client 上发生了拆包，Server 收到 pkg1 和 pkg2 后就要进行粘包处理；

4）Server 分两次读到了两个独立的数据包，分别是 pkg1 和 pkg2，其中 pkg1 包含了一部分 data1、pkg2 包含了另一部分 data1 和完整的 data2。也就是说 Client 上发生了拆包，Server 收到 pkg1 和 pkg2 后就要进行粘包处理。


![](https://img-blog.csdnimg.cn/img_convert/06d0badea2cb4aa5e16b1bed741521f3.png)


下面给出一个基于 Netty 的客户端服务器通信程序来验证这种现象的存在。

服务端代码为：

```java
/**
 * 需要的依赖：
 * <dependency>
 * <groupId>io.netty</groupId>
 * <artifactId>netty-all</artifactId>
 * <version>4.1.54.Final</version>
 * </dependency>
 */
public static void main(String[] args) throws InterruptedException {
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();

    try {
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap
            .group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, 128)
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childHandler(
            new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(
                    SocketChannel socketChannel
                ) {
                    socketChannel.pipeline().addLast(
                        new NettyServerHandler()
                    );
                }
            }
        );

        System.out.println("server is ready...");
        ChannelFuture channelFuture = bootstrap.bind(8080).sync();
        channelFuture.channel().closeFuture().sync();
    } finally {
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}

static class NettyServerHandler extends SimpleChannelInboundHandler<ByteBuf> {
    // 记录 read 事件处理次数
    private int count;

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
        byte[] buffer = new byte[msg.readableBytes()];
        msg.readBytes(buffer);

        // 将 buffer 转成字符串
        String message = new String(buffer, CharsetUtil.UTF_8);
        System.out.println("data received: " + message);
        System.out.println("data reception time: " + (++count));

        // 服务器回送一个 UUID 给客户端
        ctx.writeAndFlush(
            Unpooled.copiedBuffer(
                "\n" + UUID.randomUUID().toString(),
                CharsetUtil.UTF_8
            )
        );
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
        throws Exception {
        // 关闭与服务器端的 Socket 连接
        cause.printStackTrace();
        ctx.channel().close();
    }
}
```

客户端端代码为：

```java
/**
 * 需要的依赖：
 * <dependency>
 * <groupId>io.netty</groupId>
 * <artifactId>netty-all</artifactId>
 * <version>4.1.54.Final</version>
 * </dependency>
 */
public static void main(String[] args) throws InterruptedException {
    EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

    try {
        Bootstrap bootstrap = new Bootstrap();
        bootstrap
            .group(eventLoopGroup)
            .channel(NioSocketChannel.class)
            .handler(
            new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(
                    SocketChannel socketChannel
                ) {
                    socketChannel.pipeline().addLast(
                        new MyNtyClient.NettyClientHandler()
                    );
                }
            }
        );

        System.out.println("client is ready...");
        ChannelFuture channelFuture = bootstrap.connect(
            "127.0.0.1",
            8080).sync();
        channelFuture.channel().closeFuture().sync();
    } finally {
        eventLoopGroup.shutdownGracefully();
    }
}

static class NettyClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
    // 记录 read 事件处理次数
    private int count;

    @Override
    public void channelActive(ChannelHandlerContext ctx)
        throws Exception {
        // 向服务器发送多个短文本，使用 for 循环发送 10 次
        for (int i = 0; i < 10; i++) {
            ctx.writeAndFlush(
                Unpooled.copiedBuffer(
                    "\nhello server, " + i,
                    CharsetUtil.UTF_8
                )
            );
        }
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
        byte[] buffer = new byte[msg.readableBytes()];
        msg.readBytes(buffer);

        // 将 buffer 转成字符串
        java.lang.String message = new java.lang.String(buffer, CharsetUtil.UTF_8);
        System.out.println("data received: " + message);
        System.out.println("data reception time: " + (++count));
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
        throws Exception {
        // 关闭与客户端的 Socket 连接
        cause.printStackTrace();
        ctx.channel().close();
    }
}
```

以上两段代码中，客户端启动后会通过 for 循环向服务端发送 10 次“hello server+编号”，服务端收到后会将消息打印出来，并显示出接受了多少次才把 10 个“hello server+编号”接收完。并且，服务端每次收到消息后，都会给客户端回传一个 UUID 字符串。同样，客户端也会把服务端的回传消息打印出来，并显示出接受了多少次才把回传的 UUID 接收完。

先启动服务器，然后分别启动三个客户端进行三次试验，结果如下：

| 试验编号 | 服务端输出                                                   | 客户端输出                                                   |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1        | data received: <br/>hello server, 0<br/>hello server, 1<br/>hello server, 2<br/>hello server, 3<br/>hello server, 4<br/>hello server, 5<br/>hello server, 6<br/>hello server, 7<br/>hello server, 8<br/>hello server, 9<br/>data reception time: 1 | data received: <br/>822ba7f1-96ba-4d76-ae71-e1cb53b2e061<br/>data reception time: 1 |
| 2        | data received: <br/>hello server, 0<br/>data reception time: 1<br/>data received: <br/>hello server, 1<br/>data reception time: 2<br/>data received: <br/>hello server, 2<br/>hello server, 3<br/>hello server, 4<br/>hello server, 5<br/>hello server, 6<br/>hello server, 7<br/>hello server, 8<br/>hello server, 9<br/>data reception time: 3 | data received: <br/>af0d96ed-1941-47ed-8a8a-c71e38d14a50<br/>eb3b378c-a13c-4040-b3b3-c9dd05ba3798<br/>569e728e-0dd1-4e48-9081-4674adb72d4a<br/>data reception time: 1 |
| 3        | data received: <br/>hello server, 0<br/>data reception time: 1<br/>data received: <br/>hello server, 1<br/>data reception time: 2<br/>data received: <br/>hello server, 2<br/>hello server, 3<br/>hello server, 4<br/>hello server, 5<br/>data reception time: 3<br/>data received: <br/>hello server, 6<br/>hello server, 7<br/>data reception time: 4<br/>data received: <br/>hello server, 8<br/>hello server, 9<br/>data reception time: 5 | data received: <br/>0c857ede-ffc2-461c-b135-c72c5e809617<br/>ec2762e6-5756-4217-8715-590928a537f1<br/>031480a5-8c66-42a2-a030-8bacd1821716<br/>ab911426-7bbc-4ce4-9a0d-70bacd1c6a67<br/>06743f5b-481f-4db4-b086-038ddf5e1dbf<br/>data reception time: 1 |

可以看到三次试验中，都出现了粘包现象，即客户端的发送代码：

```java
static class NettyClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
    // 记录 read 事件处理次数
    private int count;
    
    @Override
    public void channelActive(ChannelHandlerContext ctx)
        throws Exception {
        // 向服务器发送多个短文本，使用 for 循环发送 10 次
        for (int i = 0; i < 10; i++) {
            ctx.writeAndFlush(
                Unpooled.copiedBuffer(
                    "\nhello server, " + i,
                    CharsetUtil.UTF_8
                )
            );
        }
    }
......
```
的意图是分 10 个包把 10 个“hello server+编号”发给服务器，但发出的包尺寸很小，间隔时间又很短，于是客户端做了粘包处理，用少于 10 个包把 10 个“hello server+编号”发给服务器。同样，服务端回传 UUID 也存在粘包过程。

我们把客户端的发送代码做以下改写：

```java
static class NettyClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
    // 记录 read 事件处理次数
    private int count;
    
    @Override
    public void channelActive(ChannelHandlerContext ctx)
            throws Exception {
        StringBuilder sb = new StringBuilder("");
        for (int i = 0; i < 10000; i++) {
            sb.append("\nhello server, " + i);
        }

        // 向服务器一次发送一个超长文本
        ctx.writeAndFlush(
                Unpooled.copiedBuffer(
                        sb.toString(),
                        CharsetUtil.UTF_8
                )
        );
    }
......
```

再次启动客户端，结果如下：

| 试验编号 | 服务端输出                                                   | 客户端输出                                                   |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 4        | data received: <br/>hello server, 0<br/>hello server, 1<br/>......<br/>hello server, 59<br/>hello server,<br/>data reception time: 1<br/>data received:  60<br/>hello server, 61<br/>hello server, 62<br/>......<br/>hello server, 972<br/>hel<br/>data reception time: 2<br/>data received: lo server, 973<br/>hello server, 974<br/>hello server, 975<br/>......<br/>hello server, 4422<br/>hello server, 44<br/>data reception time: 3<br/>data received: 23<br/>hello server, 4424<br/>hello server, 4425<br/>......<br/>hello server, 7872<br/>he<br/>data reception time: 4<br/>data received: llo server, 7873<br/>hello server, 7874<br/>hello server, 7875<br/>......<br/>hello server, 9999<br/>data reception time: 5 | data received: <br/>24a735eb-6f92-4301-a5d7-6d62c060d86c<br/>data reception time: 1<br/>data received: <br/>b331fa45-3ed0-4f70-8546-92d1a30fd657<br/>data reception time: 2<br/>data received: <br/>9ca0009f-cb37-4a6d-9d24-048820f6b788<br/>data reception time: 3<br/>data received: <br/>59a9b26d-b774-440d-a8e4-edf3257d0e89<br/>data reception time: 4<br/>data received: <br/>88fdb504-791e-416c-a162-74c1c8a78297<br/>data reception time: 5 |

可以发现，客户端意欲一次性发送的超长文本，由于数据量太大，被拆成了 5 次发送，这就是拆包现象。

TCP 层进行的拆包与粘包“并不顾及应用层的感受”。例如从下面这段输出可以看到，有一个“hello”被拆成了两段放入了两个包中。这就是全文所说的“面向流的通信是无消息边界保护的，因此接收端难以分辨出完整的数据包”的含义。

```txt
......
hello server, 971
hello server, 972
hel
data reception time: 2
data received: lo server, 973
hello server, 974
......
```

因此，这就对接收方提出了一个要求：接收方一定要区分每一段完整的数据，然后才能进行数据处理。例如，一个 bean 被序列化成 Json 字符串之后进行传输，若该字符串很长，被拆包传输，那么接收方一定要接收到完整的 Json 字符串之后才能进行解析，否则会出现数据处理异常。

### 2.2. Tcp 粘包拆包的解决方案

在了解了 Tcp 粘包拆包给应用层数据带来的麻烦之后，下面给出解决这种麻烦的方案。

解决这个麻烦的关键就是要让服务器每次读取数据时都能清楚每一段应用数据的边界，即能够识别每段应用数据的长度。一旦这个问题得到了解决，就不会出现服务器读取应用数据时多度或者少读的问题，从而避免 TCP 的粘包拆包机制带来的麻烦。通常使用自定义协议 + 编解码器来解决这一问题。

依然基于前面的客户端服务器代码，我们定义一种消息协议 MyProtocolMessage，并给出编码器和解码器如下：

```java
/**
 * 私有协议消息定义
 */
class MyProtocolMessage {
    private int len;
    private byte[] content;

    // 省略 getter 和 setter
}

/**
 * 编码器，MessageToByteEncoder 是 ChannelOutboundHandler 的实现类，因此是一个 OutboundHandler
 */
class MyMessageEncoder extends MessageToByteEncoder<MyProtocolMessage> {

    @Override
    protected void encode(
            ChannelHandlerContext ctx,
            MyProtocolMessage msg,
            ByteBuf out) throws Exception {
        // 把 MyProtocolMessage 编码成字节
        out.writeInt(msg.getLen());
        out.writeBytes(msg.getContent());
    }
}

/**
 * 解码器，ReplayingDecoder 是 ChannelInboundHandler 的实现类，因此是一个 InboundHandler
 */
class MyMessageDecoder extends ReplayingDecoder<Void> {

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
        // 将收到的字节解码成 MyProtocolMessage
        int length = in.readInt();
        byte[] content = new byte[length];
        in.readBytes(content);

        MyProtocolMessage myProtocolMessage = new MyProtocolMessage();
        myProtocolMessage.setLen(length);
        myProtocolMessage.setContent(content);

        // 放入 out，传递给下一个 Handler
        out.add(myProtocolMessage);
    }
}
```

这是一种基于长度的私有协议，即协议使用了一个 len 字段来规定一个协议体的长度。有了以上的私有协议+编解码器，服务端的代码可以改写如下：

```java
public static void main(String[] args) throws InterruptedException {
    EventLoopGroup bossGroup = new NioEventLoopGroup();
    EventLoopGroup workerGroup = new NioEventLoopGroup();

    try {
        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap
            .group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, 128)
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childHandler(
            new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(
                    SocketChannel socketChannel
                ) {
                    // 注意这里，先添加 MyMessageDecoder 和 MyMessageEncoder，
                    // 再添加 NettyServerHandler
                    socketChannel.pipeline().addLast(
                        // 解码 Handler
                        new MyMessageDecoder()
                    ).addLast(
                        // 编码 Handler
                        new MyMessageEncoder()
                    ).addLast(
                        // 业务处理 Handler
                        new NettyServerHandler()
                    );
                }
            }
        );

        System.out.println("server is ready...");
        ChannelFuture channelFuture = bootstrap.bind(8080).sync();
        channelFuture.channel().closeFuture().sync();
    } finally {
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
    }
}

static class NettyServerHandler 
    extends SimpleChannelInboundHandler<MyProtocolMessage> {
    // 记录 read 事件处理次数
    private int count;

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, MyProtocolMessage msg) 
        throws Exception {

        // 接收数据并处理
        int length = msg.getLen();
        byte[] content = msg.getContent();

        // 将 content 转成字符串
        String message = new String(content, CharsetUtil.UTF_8);
        System.out.println("data received: " + message + "\ndata size (byte): " + length);
        System.out.println("data reception time: " + (++count));

        // 服务器回送一个 UUID 给客户端
        String uuid = "\n" + UUID.randomUUID().toString();
        byte[] content2 = uuid.getBytes(CharsetUtil.UTF_8);
        int length2 = content2.length;
        MyProtocolMessage myProtocolMessage = new MyProtocolMessage();
        myProtocolMessage.setLen(length2);
        myProtocolMessage.setContent(content2);
        ctx.writeAndFlush(myProtocolMessage);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {
        // 关闭与服务器端的 Socket 连接
        cause.printStackTrace();
        ctx.channel().close();
    }
}
```

客户端的代码可以改写如下：

```java
public static void main(String[] args) throws InterruptedException {
    EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

    try {
        Bootstrap bootstrap = new Bootstrap();
        bootstrap
            .group(eventLoopGroup)
            .channel(NioSocketChannel.class)
            .handler(
            new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(
                    SocketChannel socketChannel
                ) {
                    // 注意这里，先添加 MyMessageDecoder 和 MyMessageEncoder，
                    // 再添加 NettyClientHandler
                    socketChannel.pipeline().addLast(
                        // 解码 Handler
                        new MyMessageDecoder()
                    ).addLast(
                        // 编码 Handler
                        new MyMessageEncoder()
                    ).addLast(
                        // 业务处理 Handler
                        new NettyClientHandler()
                    );
                }
            }
        );

        System.out.println("client is ready...");
        ChannelFuture channelFuture = bootstrap.connect(
                "127.0.0.1",
                8080).sync();
        channelFuture.channel().closeFuture().sync();
    } finally {
        eventLoopGroup.shutdownGracefully();
    }
}

static class NettyClientHandler 
    extends SimpleChannelInboundHandler<MyProtocolMessage> {
    // 记录 read 事件处理次数
    private int count;

    @Override
    public void channelActive(ChannelHandlerContext ctx)
            throws Exception {

        // 向服务器发送多个短文本，使用 for 循环发送 10 次
        for (int i = 0; i < 10; i++) {
            String str = "\nhello server, " + i;
            byte[] content = str.getBytes(CharsetUtil.UTF_8);
            int length = content.length;

            // 创建协议包对象
            MyProtocolMessage myProtocolMessage = new MyProtocolMessage();
            myProtocolMessage.setLen(length);
            myProtocolMessage.setContent(content);
            ctx.writeAndFlush(myProtocolMessage);
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {
        // 关闭与客户端的 Socket 连接
        cause.printStackTrace();
        ctx.channel().close();
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, MyProtocolMessage msg) 
        throws Exception {
        // 接收数据并处理
        int length = msg.getLen();
        byte[] content = msg.getContent();

        // 将 content 转成字符串
        String message = new String(content, CharsetUtil.UTF_8);
        System.out.println("data received: " + message + "\ndata size (byte): " + length);
        System.out.println("data reception time: " + (++count));
    }
}
```

以上服务器和客户端代码的改写思路是：

1）向对方发送消息之前先编码

2）接收对方消息后先解码再处理

3）发送方遵守私有协议，先发送消息长度，再发送消息内容

```java
out.writeInt(msg.getLen());
out.writeBytes(msg.getContent());
```

4）接收方遵守私有协议，先接收消息长度，再接收该长度的字节数作为消息内容

```java
int length = in.readInt();
byte[] content = new byte[length];
in.readBytes(content);
```

先启动服务器，然后分别启动三个客户端进行三次试验，结果如下：

| 试验编号 | 服务端输出                                                   | 客户端输出                                                   |
| :------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1、2、3  | data received: <br/>hello server, 0<br/>data size (byte): 16<br/>data reception time: 1<br/>data received: <br/>hello server, 1<br/>data size (byte): 16<br/>data reception time: 2<br/>data received: <br/>hello server, 2<br/>data size (byte): 16<br/>data reception time: 3<br/>data received: <br/>hello server, 3<br/>data size (byte): 16<br/>data reception time: 4<br/>data received: <br/>hello server, 4<br/>data size (byte): 16<br/>data reception time: 5<br/>data received: <br/>hello server, 5<br/>data size (byte): 16<br/>data reception time: 6<br/>data received: <br/>hello server, 6<br/>data size (byte): 16<br/>data reception time: 7<br/>data received: <br/>hello server, 7<br/>data size (byte): 16<br/>data reception time: 8<br/>data received: <br/>hello server, 8<br/>data size (byte): 16<br/>data reception time: 9<br/>data received: <br/>hello server, 9<br/>data size (byte): 16<br/>data reception time: 10 | data received: <br/>0687b651-2c4e-4f17-8eed-048080a85997<br/>data size (byte): 37<br/>data reception time: 1<br/>data received: <br/>f2c2a37a-532f-4e78-b046-a520dee03b22<br/>data size (byte): 37<br/>data reception time: 2<br/>data received: <br/>49009386-0414-44a4-9aee-d6072afc5cbf<br/>data size (byte): 37<br/>data reception time: 3<br/>data received: <br/>6296b3a7-805d-4cf3-9c88-9135d3d48ada<br/>data size (byte): 37<br/>data reception time: 4<br/>data received: <br/>fbc3d723-d3ee-4077-9562-1ac7079dda73<br/>data size (byte): 37<br/>data reception time: 5<br/>data received: <br/>6a685424-68fd-4959-b46b-5a3acc3601d7<br/>data size (byte): 37<br/>data reception time: 6<br/>data received: <br/>59aeb741-b2b1-461e-ac95-dc72e08f49f4<br/>data size (byte): 37<br/>data reception time: 7<br/>data received: <br/>a0d1225b-770c-48ae-a1bc-2e1918848e49<br/>data size (byte): 37<br/>data reception time: 8<br/>data received: <br/>748fa17c-af5e-49eb-9d2b-697d32b2db3d<br/>data size (byte): 37<br/>data reception time: 9<br/>data received: <br/>810fc904-f936-41be-b655-8f2341c4653f<br/>data size (byte): 37<br/>data reception time: 10 |

可以发现，粘包现象已经被解决了。

同样，若将客户端代码修改为一次性向服务器发送一个超长文本，也不会再发生拆包现象。代码修改如下：

```java
static class NettyClientHandler extends SimpleChannelInboundHandler<MyProtocolMessage> {
    // 记录 read 事件处理次数
    private int count;

    @Override
    public void channelActive(ChannelHandlerContext ctx)
            throws Exception {

        // 向服务器一次发送一个超长文本
        StringBuilder sb = new StringBuilder("");
        for (int i = 0; i < 10000; i++) {
            sb.append("\nhello server, " + i);
        }

        byte[] bytes = sb.toString().getBytes(CharsetUtil.UTF_8);
        // 创建协议包对象
        MyProtocolMessage myProtocolMessage = new MyProtocolMessage();
        myProtocolMessage.setLen(bytes.length);
        myProtocolMessage.setContent(bytes);
        ctx.writeAndFlush(myProtocolMessage);
    }
    ......
```

试验结果为：

| 试验编号 | 服务端输出                                                   | 客户端输出                                                   |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 4、5、6  | data received: <br/>hello server, 0<br/>hello server, 1<br/>......<br/>hello server, 9998<br/>hello server, 9999<br/>data size (byte): 188890<br/>data reception time: 1 | data received: <br/>b37905cb-bca0-4067-ba08-3df267e4b6eb<br/>data size (byte): 37<br/>data reception time: 1 |

## 3. 通过 SslHandler 加密传输数据

为了保护传输层的数据，我们通常会使用像 SSL 和 TLS 这样的安全协议，它们层叠在其他协议之上，用以实现数据安全。为了支持 SSL/TLS，Java 提供了 javax.net.ssl 包，它的 SSLContext 和 SSLEngine 类使得实现解密和加密相当简单直接。Netty 通过一个名为 SslHandler 的 ChannelHandler 实现类利用了 JDK 的这些 API，SslHandler 在内部使用 SSLEngine 来完成实际的工作。

Netty 还提供了基于 OpenSSL 工具包的 OpenSslEngine 类，它比 JDK 提供的 SSLEngine 类有更好的性能。Netty 允许使用者进行配置，但无论是使用 JDK 的 SSLEngine 还是使用 Netty 的 OpenSslEngine，SSL API 和数据流都是一致的。下图展示了使用 SslHandler 的数据流。


![](https://img-blog.csdnimg.cn/img_convert/7a5ad65174e8d29924e631bc225e486e.png)


在大多数情况下，SslHandler 将是 ChannelPipeline 中的第一个 ChannelHandler。这确保了只有在所有其他的 ChannelHandler 将它们的逻辑应用到数据之后，才会进行加密。体现在代码上如下：

```java
// 前文都是使用 ChannelInitializer 的匿名内部类创建 ChannelInitializer 的实例，
// 也可以像下面这样定义一个 SslChannelInitializer，然后创建其实例。
public class SslChannelInitializer extends ChannelInitializer<Channel>{
    private final SslContext context;
    private final boolean startTls;
    public SslChannelInitializer(
        SslContext context, boolean startTls) {
        this.context = context;
        this.startTls = startTls;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        SSLEngine engine = context.newEngine(ch.alloc());
        ch.pipeline().addFirst(
            "ssl",
            new SslHandler(engine, startTls)
        );
        
        // 再往 pipeline 中添加其他的 Handler
    }
}
```

SslHandler 提供了一些有用的方法，如下表所示。

| 方法名称                                                     | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| handshakeFuture()                                            | 返回一个在握手完成后将会得到通知的 ChannelFuture 。如果握手先前已经执行过了，则返回一个包含了先前的握手结果的 ChannelFuture |
| setHandshakeTimeout (long,TimeUnit)<br/>setHandshakeTimeoutMillis (long)<br/>getHandshakeTimeoutMillis() | 设置和获取握手的超时时间，超时之后，握手 ChannelFuture 将会被通知失败 |
| close()<br/>close(ChannelPromise)<br/>close(ChannelHandlerContext,ChannelPromise) | 发送 close_notify 以请求关闭并销毁底层的 SslEngine           |
| setCloseNotifyTimeout (long,TimeUnit)<br/>setCloseNotifyTimeoutMillis (long)<br/>getCloseNotifyTimeoutMillis() | 设置和获取发送 close_notify 的超时时间，超时之后，连接将会被强制关闭 |

例如，在握手阶段，两个节点将相互验证并且商定一种加密方式。使用者可以通过配置 SslHandler 来修改它的行为，或者在 SSL/TLS 握手一旦完成之后提供通知。SSL/TLS 握手将会被自动执行，握手阶段完成之后，所有的数据都将会被加密。

## 4. Netty 的编解码器机制

在上面的粘包拆包处理的示例中，使用到了编码器和解码器。

在编写网络应用程序时，要考虑到数据在网络中都是以二进制字节码进行传输的，因此在发送数据前要对应用数据（业务数据）进行编码，在接收到数据时，要对数据进行解码。


![](https://img-blog.csdnimg.cn/img_convert/cd4aa2f405c13faf803829d9b140c21b.png)


Netty 对编解码也提供了支持，位于 io.netty.handler.codec 下面，包括编码器 Handler 和解码器 Handler。


![](https://img-blog.csdnimg.cn/img_convert/c10bb5995094953dcf264d253cd5a11e.png)


从源码包中包含的内容可以看出，Netty 提供的编码器和解码器对应用层的一些公私有协议进行了支持。例如：

1）上文使用的 MessageToByteEncoder 和 ReplayingDecoder 就实现了二进制字节流和 MyProtocolMessage 之间的互相转换

2）http、http2 包下面的的编解码器对 http 协议提供了支持

3）protobuf 包下面的编解码器对基于 Google Protobuf 结构化数据传输提供了支持

4）xml 包下面的编解码器对基于 xml 结构化数据传输提供了支持

5）redis 包下面的编解码器对 redis 协议提供了支持

......

## 5. Netty 对 HTTP/HTTPS 的支持

HTTP/HTTPS 是基于请求/响应模式的：客户端向服务器发送一个 HTTP/HTTPS 请求，然后服务器将会返回一个 HTTP/HTTPS 响应。对于 HTTP/HTTPS 协议的细节，本文不做描述。

在 Netty 中，Http 请求被抽象为 HttpRequest 对象+多个 HttpContent 对象+LastHttpContent 对象，如下图所示：


![](https://img-blog.csdnimg.cn/img_convert/d2a8201e1a2e1875741144f7a13f43c1.png)


同样，Http 响应被抽象为 HttpResponse 对象+多个 HttpContent 对象+LastHttpContent 对象，如下图所示：


![](https://img-blog.csdnimg.cn/img_convert/1850091ba1f807b18fbdda838c0a9622.png)


以上 HttpRequest、HttpResponse、HttpContent、LastHttpContent 类均继承于 HttpObject 类。

基于以上抽象，Netty 提供了多种编码器和解码器以简化对这个协议的使用。Netty 内置的 HTTP 编码器和编解码器如下表所示：

| 名称                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| HttpRequestEncoder  | 将 HttpRequest 、 HttpContent 和 LastHttpContent 消息编码为字节 |
| HttpResponseEncoder | 将 HttpResponse 、 HttpContent 和 LastHttpContent 消息编码为字节 |
| HttpRequestDecoder  | 将字节解码为 HttpRequest 、 HttpContent 和 LastHttpContent 消息 |
| HttpResponseDecoder | 将字节解码为 HttpResponse 、 HttpContent 和 LastHttpContent 消息 |

它们的使用也非常简单，例如：

```java
public class HttpPipelineInitializer extends ChannelInitializer<Channel> {
    private final boolean client;

    public HttpPipelineInitializer(boolean client) {
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (client) {
            // 客户端要编码请求、解码响应
            pipeline.addLast("decoder", new HttpResponseDecoder());
            pipeline.addLast("encoder", new HttpRequestEncoder());
        } else {
            // 服务端要解码请求、编码响应
            pipeline.addLast("decoder", new HttpRequestDecoder());
            pipeline.addLast("encoder", new HttpResponseEncoder());
        }
    }
}
```

由于 HTTP 的请求和响应可能由许多部分组成，因此 Netty 使用者需要聚合它们以形成完整的消息。为了消除这项繁琐的任务，Netty 提供了一个聚合器，它可以将多个消息部分合并为 FullHttpRequest 或者 FullHttpResponse 消息。通过这样的方式，使用者将总是能接收到完整的消息内容。由于消息分段需要被缓冲，直到可以转发一个完整的消息给下一个 ChannelInboundHandler，所以这个操作有轻微的时间开销，其所带来的好处是使用者不必关心消息碎片了。

引入这种自动聚合机制也很简单，就是是向 ChannelPipeline 中额外添加一个 HttpObjectAggregator。代码如下：

```java
public class HttpAggregatorInitializer extends ChannelInitializer<Channel> {
    private final boolean isClient;

    public HttpAggregatorInitializer(boolean isClient) {
        this.isClient = isClient;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (isClient) {
            // HttpClientCodec 具备编解码处理能力，相当于
            // HttpResponseDecoder + HttpRequestEncoder
            pipeline.addLast("codec", new HttpClientCodec());
        } else {
            // HttpServerCodec 具备编解码处理能力，相当于
            // HttpRequestDecoder + HttpResponseEncoder
            pipeline.addLast("codec", new HttpServerCodec());
        }
        
        // 将最大的消息大小为 512 KB 的 HttpObjectAggregator 
        // 添加到 ChannelPipeline
        pipeline.addLast(
            "aggregator",
            new HttpObjectAggregator(512 * 1024)
        );
    }
}
```

若要基于 Netty 开发 HTTPS 应用，也很简单，就是往 pipline 中添加 SslHandler 以及 Http 编解码 Handler（Https 协议本就是在 Http 协议的基础上添加了 Ssl），代码示例如下：

```java
public class HttpsCodecInitializer extends ChannelInitializer<Channel> {
    private final SslContext context;
    private final boolean isClient;

    public HttpsCodecInitializer(SslContext context, boolean isClient) {
        this.context = context;
        this.isClient = isClient;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        SSLEngine engine = context.newEngine(ch.alloc());
        
        // 添加 SslHandler
        pipeline.addFirst("ssl", new SslHandler(engine));
        
        // 添加 Http 协议编解码 Handler
        if (isClient) {
            pipeline.addLast("codec", new HttpClientCodec());
        } else {
            pipeline.addLast("codec", new HttpServerCodec());
        }
    }
}
```

## 6. Netty 对 WebSocket 的支持

WebSocket 是一种在 2011 年被互联网工程任务组（IETF）标准化的协议，WebSocket 旨在为 Web 上的双向数据传输问题提供一个切实可行的解决方案，使得客户端和服务器之间可以在任意时刻传输消息，因此，这也就要求它们异步地处理消息回执。

WebSocket 解决了一个长期存在的问题：既然底层的协议（HTTP）是一个请求/响应模式的交互序列，那么如何实时地发布信息呢？AJAX 提供了一定程度上的改善，但是数据流仍然是由客户端所发送的请求驱动的。WebSocket 在客户端和服务器之间提供了真正的双向数据交换。尽管最早的实现仅限于文本数据，但是现在已经不是问题了，WebSocket 现在可以用于传输任意类型的数据，很像普通的套接字。

下给出了 WebSocket 协议的一般概念。在这个场景下，通信将以普通的 HTTP 协议开始，随后升级到双向的 WebSocket 协议。


![](https://img-blog.csdnimg.cn/img_convert/f0fd153d421fbdefaa91779b3e76137c.png)


由于应用层协议的原理不是本文的重点，本文不再对 websocket 做更深入的描述。

Netty 对于 WebSocket 协议也提供了支持，使用起来也很简单，且无需关心它内部的实现细节。要想向应用程序中添加对于 WebSocket 的支持，只需要将适当的客户端或者服务器 WebSocket ChannelHandler 添加到 ChannelPipeline 中。这个类将处理由 WebSocket 定义的称为帧的特殊消息类型。如下表所示，WebSocketFrame 可以被归类为数据帧或者控制帧。

| 名称                       | 描述                                                         |
| -------------------------- | ------------------------------------------------------------ |
| BinaryWebSocketFrame       | 数据帧：二进制数据                                           |
| TextWebSocketFrame         | 数据帧：文本数据                                             |
| ContinuationWebSocketFrame | 数据帧：属于上一个 BinaryWebSocketFrame 或者 TextWebSocketFrame 的二进制或者文本数据 |
| CloseWebSocketFrame        | 控制帧：用于关闭 WebSocket 连接                                |
| PingWebSocketFrame         | 控制帧：请求一个 PongWebSocketFrame                          |
| PongWebSocketFrame         | 控制帧：对 PingWebSocketFrame 请求的响应                     |

因为 Netty 主要是一种服务器端的技术，下面重点给出创建 WebSocket 服务器的代码。代码展示了一个使用 WebSocketServerProtocolHandler 的简单示例，这个类处理协议升级握手，以及 3 种控制帧——Close、Ping 和 Pong。而 Text 和 Binary 数据帧将会被传递给下一个（由使用者实现的）ChannelHandler 进行处理。

```java
public class WebSocketServerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        // 如果要想为 WebSocket 添加安全性，只需要将 SslHandler 作
        // 为第一个 ChannelHandler 添加到 ChannelPipeline 中
        ch.pipeline().addLast(
                new HttpServerCodec(),
                new HttpObjectAggregator(65536),
                new WebSocketServerProtocolHandler("/websocket"),
                new TextFrameHandler(),
                new BinaryFrameHandler(),
                new ContinuationFrameHandler());
    }

    public final class TextFrameHandler extends
            SimpleChannelInboundHandler<TextWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
                                 TextWebSocketFrame msg) throws Exception {
            // 自行实现处理 text frame 的逻辑 
        }
    }

    public final class BinaryFrameHandler extends
            SimpleChannelInboundHandler<BinaryWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
                                 BinaryWebSocketFrame msg) throws Exception {
            // 自行实现处理 binary frame 的逻辑 
        }
    }

    public final class ContinuationFrameHandler extends
            SimpleChannelInboundHandler<ContinuationWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx,
                                 ContinuationWebSocketFrame msg) throws Exception {
            // 自行实现处理 continuation frame 的逻辑 
        }
    }
}
```

检测空闲连接以及超时对于及时释放资源来说是至关重要的。这在 Socket 和 WebSocket 编程中是一个常见的工作，Netty 特地提供了几个 ChannelHandler 实现连接管理，如下表所示：

| 名称                | 描述                                                         |
| ------------------- | ------------------------------------------------------------ |
| IdleStateHandler    | 当连接空闲时间太长时，将会触发一个 IdleStateEvent 事件。然后，可以通过在 ChannelInboundHandler 中重写 userEventTriggered() 方法来处理该 IdleStateEvent 事件 |
| ReadTimeoutHandler  | 如果在指定的时间间隔内没有收到任何的入站数据，则抛出一个 ReadTimeoutException 并关闭对应的 Channel 。可以通过重写 ChannelHandler 中的 exceptionCaught() 方法来检测该 ReadTimeoutException |
| WriteTimeoutHandler | 如果在指定的时间间隔内没有任何出站数据写入，则抛出一个 WriteTimeoutException 并关闭对应的 Channel 。可以通过重写 ChannelHandler 中的 exceptionCaught() 方法检测该 WriteTimeoutException |

下面的代码展示了在实践中使用得最多的 IdleStateHandler 的使用示例。

```java
public class IdleStateHandlerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(
            // 如果在 60 秒之内没有接收或者发送任何的数据，
            // 我们将得到通知
            new IdleStateHandler(0, 0, 60, TimeUnit.SECONDS)
        );
        pipeline.addLast(new HeartbeatHandler());
    }

    public final class HeartbeatHandler
            extends ChannelInboundHandlerAdapter {
        private final ByteBuf HEARTBEAT_SEQUENCE =
                Unpooled.unreleasableBuffer(
                        Unpooled.copiedBuffer("HEARTBEAT", CharsetUtil.ISO_8859_1)
                );

        @Override
        public void userEventTriggered(ChannelHandlerContext ctx,
                                       Object evt) throws Exception {
            if (evt instanceof IdleStateEvent) {
                ctx.writeAndFlush(HEARTBEAT_SEQUENCE.duplicate())
                        .addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                super.userEventTriggered(ctx, evt);
            }
        }
    }
}
```

这个示例演示了如何使用 IdleStateHandler 来测试远程节点是否仍然还活着，并且在它失活时通过关闭连接来释放资源。如果连接上超过 60 秒没有接收或者发送任何的数据，那么 IdleStateHandler 将会使用一个 IdleStateEvent 事件来调用 fireUserEventTriggered()方法。HeartbeatHandler 实现了 userEventTriggered()方法，如果这个方法检测到 IdleStateEvent 事件，它将会发送心跳消息，并且添加一个将在发送操作失败时关闭该连接的 ChannelFutureListener 。

## 7. Netty 对 UDP 的支持

UDP 协议对 IP 协议进行了简单的封装，没有提供连接管理、可靠传输、流量控制、拥塞控制等机制。因此 UDP 具有资源消耗小、处理速度快等优点。通常即时通信系统和流媒体系统（容忍一定程度的数据的不可靠传输、但追求传输速度）往往会采用 UDP 协议来实现数据传输，但总的来说 TCP 在互联网世界的应用更多。

本文对 UDP 协议不做过多的描述，下面给出基于 Netty 进行 UDP 应用开发的实例。对代码的解析放在了注释中。

UDP 服务端代码：

```java
......

public class UdpDemoServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.group(group)
                    // 由于是使用 UDP 协议进行通信，因此服务端使用的是 NioDatagramChannel
                    .channel(NioDatagramChannel.class)
                    // 支持 UDP 广播
                    .option(ChannelOption.SO_BROADCAST, true)
                    // 设置业务处理 Handler。由于 UDP 服务器与客户端之间不存在实际的连接，因
                    // 此不需要调用 Bootstrap#childHandler()为 ChannelPipeline 设置 handler
                    .handler(new UdpServerHandler());
            System.out.println("udp server is ready...");
            bootstrap.bind(8080).sync().channel().closeFuture().await();
        } finally {
            group.shutdownGracefully();
        }

    }

    static class UdpServerHandler extends SimpleChannelInboundHandler<DatagramPacket> {

        /**
         * Netty 对 UDP 进行了封装，因此，接收到的是 Netty 封装后的 DatagramPacket 对象
         *
         * @param ctx
         * @param msg
         * @throws Exception
         */
        @Override
        protected void channelRead0(ChannelHandlerContext ctx, DatagramPacket msg) throws Exception {
            // msg.content()返回 ByteBuf 对象
            String dataFromClient = msg.content().toString(CharsetUtil.UTF_8);
            System.out.println("data from udp client: " + dataFromClient);

            ctx.writeAndFlush(
                    // DatagramPacket 的构造方法有两个参数：
                    // 1. 第一个是要发送的内容，ByteBuf 类型
                    // 2. 第二个是目的地址，包括 IP 和端口号，可以直接从发送报文中获取
                    new DatagramPacket(
                            Unpooled.copiedBuffer(
                                    "hello client, i have received your data.",
                                    CharsetUtil.UTF_8
                            ),
                            msg.sender()
                    )
            );
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
                throws Exception {
            // 关闭 channel
            ctx.channel().close();
        }

    }
}
```

UDP 客户端代码：

```java
......

public class UdpDemoClient {
    public static void main(String[] args) throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.group(group)
                    // 由于是使用 UDP 协议进行通信，因此客户端使用的是 NioDatagramChannel
                    .channel(NioDatagramChannel.class)
                    // 支持 UDP 广播
                    .option(ChannelOption.SO_BROADCAST, true)
                    // 设置业务处理 Handler
                    .handler(new UdpClientHandler());
            System.out.println("udp client is ready...");
            Channel channel = bootstrap.bind(0).sync().channel();

            // 客户端启动之后向内网段内所有的主机广播 UDP 消息
            channel.writeAndFlush(
                    new DatagramPacket(
                            Unpooled.copiedBuffer(
                                    "hello! respond me if you are udp server.",
                                    CharsetUtil.UTF_8
                            ),
                            new InetSocketAddress("255.255.255.255", 8080)
                    )
            ).sync();

            // 如果 15s 内未收到内网中 UDP 服务器的回复，关闭客户端
            if (!channel.closeFuture().await(15000)) {
                System.out.println("timeout! no udp server response received");
            }
        } finally {
            group.shutdownGracefully();
        }

    }

    static class UdpClientHandler extends SimpleChannelInboundHandler<DatagramPacket> {

        /**
         * Netty 对 UDP 进行了封装，因此，接收到的是 Netty 封装后的 DatagramPacket 对象
         *
         * @param ctx
         * @param msg
         * @throws Exception
         */
        @Override
        protected void channelRead0(ChannelHandlerContext ctx, DatagramPacket msg) throws Exception {
            // msg.content()返回 ByteBuf 对象
            String dataFromClient = msg.content().toString(CharsetUtil.UTF_8);
            System.out.println("find udp server in intranet! data from udp server: " + dataFromClient);
            ctx.close();
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
                throws Exception {
            // 关闭 channel
            ctx.channel().close();
        }

    }
}
```

同样，在上面的 UDP Demo 中也可以使用 Netty 提供的一系列的编解码器以实现基于 UDP 的私有应用协议，本文不再给出示例，读者可自行探索。

## 8. 基于 Netty 实现 RPC

Dubbo 是业界较为著名的 RPC 框架，Dubbo 在实现的时候采用了分层的架构，如下图示。Dubbo 在其 Transport 层使用了 Netty 来发送 Dubbo 协议消息。


![](https://img-blog.csdnimg.cn/img_convert/426ff2ffddc455401a9c07244f678c57.png)


为了帮助读者更好理解 RPC 的实现原理，以及 Dubbo 如何利用 Netty 实现应用数据传输的，本节给出一个综合性的案例：基于 Netty 实现一个简单的 RPC 过程。

### 4.1. RPC 的工作原理

远程过程调用（Remote Procedure Call），是指在分布式系统中，一个节点调用另外一个节点上的方法，就像调用本地方法一样。远程调用的实现原理如下图所示，其中调用者通常被称为 Consumer，被调用者通常被称为 Provider。


![](https://img-blog.csdnimg.cn/img_convert/e8bac4e96d858374036305422874d8dc.png)


在上图中：

1）Consumer 以本地调用方式调用 Provider 上的方法

2）ConsumerStub（相当于客户端）接收到调用请求后负责将方法、参数等封装成能够进行网络传输的消息体

3）ConsumerStub 将消息进行编码，并发送到 ProviderStub（相当于服务器）

4）ProviderStub 收到消息后进行解码

5）ProviderStub 根据解码的结果调用 Provider 上的本地的方法

6）Provider 执行本地方法，并将结果返回给 ProviderStub

7）ProviderStub 将返回值进行编码，然后发送至 ConsumerStub

8）ConsumerStub 接收到消息，进行解码

9）Consumer 得到返回值

下面给出 RPC 调用实例中的 provider 端和 consumer 端的代码。

### 4.2. provider 端的代码


![](https://img-blog.csdnimg.cn/img_convert/c7116e48774229fea48d0bec1f476a98.png)


其中 HelloService.java 的代码为：

```java
package rpc.consumer.common;

/**
 * 定义了 Consumer 和 Provider 之间的公共接口
 */
public interface HelloService {
    String hello(String msg);
}

```

HelloServiceImpl.java 的代码为：

```java
package rpc.provider;

import rpc.provider.common.HelloService;

public class HelloServiceImpl implements HelloService {
    @Override
    public String hello(String msg) {
        System.out.printf("message from consumer: " + msg);
        return "hello consumer, i have received your message!";
    }
}
```

NettyServer.java 的代码为：

```java
package rpc.provider;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

public class NettyServer {
    /**
     * 完成对 NettyServer 的初始化和启动
     */
    public static void startServer(String serviceName, String hostName, int port) {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap
                .group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(
                new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(
                        SocketChannel socketChannel
                    ) {
                        socketChannel.pipeline()
                            .addLast(
                            // 解码器 Handler
                            new StringDecoder()
                        )
                            .addLast(
                            // 编码器 Handler
                            new StringEncoder()
                        )
                            .addLast(
                            // 业务处理 Handler
                            new NettyServerHandler()
                        );
                    }
                }
            );

            System.out.println(serviceName + " is ready...");
            ChannelFuture channelFuture = bootstrap.bind(hostName, port).sync();
            channelFuture.channel().closeFuture().sync();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    static class NettyServerHandler extends ChannelInboundHandlerAdapter {

        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg)
                throws Exception {
            // 获取客户端（Consumer）发送的 RPC 调用请求消息，
            // 并据此调用服务（Provider）的本地方法
            String rpcReqData = msg.toString();

            // 我们需要定义一个 RPC 协议，来识别 consumer 对 provider 的调用。
            // 为了简便，这里通过识别“HelloService#hello#参数”格式的消
            // 息来调用 provider 的本地方法，这可以看做一个简陋的 RPC 协议
            if (rpcReqData.startsWith("HelloService#hello#")) {
                String parameter = rpcReqData.substring(rpcReqData.lastIndexOf("#") + 1);
                String returnMsg = new HelloServiceImpl().hello(parameter);
                ctx.writeAndFlush(returnMsg);
            }
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
                throws Exception {
            // 关闭与客户端的 Socket 连接
            ctx.channel().close();
        }
    }
}
```

需要说明的是，这里的简单实例仅仅用于 RPC 调用过程演示。所以使用 String 作为消息类型，且未进行拆包粘包处理。实际上，在使用 String 作为消息类型的时候，可以使用上文

使用 LineBasedFrameDecoder 来配合 StringDecoder 解决 String 数据传输中的
                            // 拆包粘包问题。LineBasedFrameDecoder
                            // 指定了 StringDelimiterBasedFrameDecoder+StringDecoder

ServerBootstrap.java 的代码为：

```java
package rpc.provider;

/**
 * ServerBootstrap 会启动一个 Provider
 */
public class ServerBootstrap {
    public static void main(String[] args) {
        NettyServer.startServer(
                "provider",
                "127.0.0.1",
                8080
        );
    }
}
```

### 4.3. consumer 端的代码


![](https://img-blog.csdnimg.cn/img_convert/88a7a86b9e77a55c680b0ac12fad5b72.png)


其中 HelloService.java 的代码为：

```java
package rpc.consumer.common;

/**
 * 定义了 Consumer 和 Provider 之间的公共接口
 */
public interface HelloService {
    String hello(String msg);
}
```

NettyClient.java 的代码为：

```java
package rpc.consumer;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.util.concurrent.*;

public class NettyClient {
    /**
     * 使用线程池完成 RPC 调用
     */
    private static ExecutorService executorService
            = new ThreadPoolExecutor(
            Runtime.getRuntime().availableProcessors(),
            Runtime.getRuntime().availableProcessors(),
            0L,
            TimeUnit.MILLISECONDS,
            new LinkedBlockingQueue<Runnable>()
    );

    private static NettyClientHandler nettyClientHandler;

    /**
     * 使用代理模式，获取一个代理对象
     */
    public Object getBean(final Class<?> serviceClass, final String protocolHeader) {
        return Proxy.newProxyInstance(
                Thread.currentThread().getContextClassLoader(),
                new Class<?>[]{serviceClass},
                new InvocationHandler() {
                    @Override
                    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                        if (nettyClientHandler == null) {
                            initClient("consumer", "127.0.0.1", 8080);
                        }

                        // 设置要发给服务器端的信息
                        nettyClientHandler.setRpcReqData(
                                // 构建自定义的 RPC 协议消息，消息格式为：“HelloService#hello#参数”
                                protocolHeader + args[0]
                        );
                        return executorService.submit(nettyClientHandler).get();
                    }
                }
        );
    }

    /**
     * 初始化客户端
     */
    public static void initClient(String serviceName, String serverHostName, int port) {
        nettyClientHandler = new NettyClientHandler();
        NioEventLoopGroup group = new NioEventLoopGroup();

        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap
                .group(group)
                .channel(NioSocketChannel.class)
                .option(ChannelOption.TCP_NODELAY, true)
                .handler(
                new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(
                        SocketChannel socketChannel
                    ) {
                        socketChannel.pipeline()
                            .addLast(
                            // 解码器 Handler
                            new StringDecoder()
                        )
                            .addLast(
                            // 编码器 Handler
                            new StringEncoder()
                        )
                            .addLast(
                            // 业务处理 Handler
                            nettyClientHandler
                        );
                    }
                }
            );

            System.out.println(serviceName + " is ready...");
            bootstrap.connect(serverHostName, port).sync();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    static class NettyClientHandler
            extends ChannelInboundHandlerAdapter
            implements Callable {

        private ChannelHandlerContext context;

        /**
         * RPC 调用的返回值
         */
        private String rpcReturnMsg;

        /**
         * RPC 调用的请求数据
         */
        private String rpcReqData;

        public void setRpcReqData(String rpcReqData) {
            this.rpcReqData = rpcReqData;
        }

        /**
         * channelActive 在客户端与服务器的连接完成之后就会被调用
         *
         * @param ctx
         * @throws Exception
         */
        @Override
        public void channelActive(ChannelHandlerContext ctx)
                throws Exception {
            // 在其他方法会用到这个 ctx，因此这里要赋值给全局变量 context
            context = ctx;
        }

        /**
         * 为该方法做多线程同步处理
         *
         * @param ctx
         * @param msg
         * @throws Exception
         */
        @Override
        public synchronized void channelRead(ChannelHandlerContext ctx, Object msg)
                throws Exception {
            // msg 是服务端返回的调用结果
            rpcReturnMsg = msg.toString();

            // 唤醒等待的线程，将 returnMsg 返回给调用者
            notify();
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
                throws Exception {
            // 关闭与服务器端的 Socket 连接
            ctx.channel().close();
        }

        /**
         * 该方法被代理对象调用，用于向服务器发送 RPC 调用请求数据，
         * 发送完了之后等待被唤醒，然后返回 RPC 调用结果
         *
         * @return
         * @throws Exception
         */
        @Override
        public synchronized Object call() throws Exception {
            // 向服务端发送 RPC 调用请求数据
            context.writeAndFlush(rpcReqData);

            // 等待 RPC 调用结果
            wait();

            // 一旦被唤醒，即说明调用结束，拿到了返回值
            return rpcReturnMsg;
        }
    }
}
```

ClientBootstrap.java 的代码为：

```java
package rpc.consumer;

import rpc.consumer.common.HelloService;

public class ClientBootsrap {
    /**
     * 定义 RPC 协议头
     */
    public static final String protocolHeader = "HelloService#hello#";

    public static void main(String[] args) {
        // 创建一个 consumer
        NettyClient consumer = new NettyClient();

        // 创建代理对象
        HelloService helloService = (HelloService) consumer
                .getBean(HelloService.class, protocolHeader);

        String res = helloService.hello("testing");
        System.out.println(res);
    }
}
```

分别启动 Provider 和 Consumer，验证调用结果如下：


![](https://img-blog.csdnimg.cn/img_convert/eaa27797e079acf5e4230b566c75e8ce.png)



![](https://img-blog.csdnimg.cn/img_convert/c4acd091aa082d3880a1c60c57df1ea2.png)


本节的 RPC 框架的实现要点为：

1）创建一个接口 HelloService，定义抽象方法 hello，用于约定 Consumer 和 Provider 之间的 RPC 调用

2）Provider 端创建一个 NettyServer，监听 Consumer 的请求，并按照自定义 RPC 协议解析消息、调用本地方法

3）Consumer 端透明地调用 HelloService#hello 方法，底层则使用 NettyClient 请求 NettyServer

## 9. Protobuf 简介

使用 Netty 实现公私有协议应用的时候。需要序列化协议消息对象，Netty 为这些对象的系列化提供了内置的编解码器，能够让使用者快速构建基于公私有应用协议的网络 IO 应用。但是 Netty 本身提供的编解码器在实现对象与二进制字节流之间的转换的时候，也存在一定的问题。例如 Netty 自带的 ObjectEncoder 和 ObjectDecoder 实现对象与二进制字节流之间的转换利用的是 Java 序列化技术，而 Java 序列化技术本身的效率就不高，而且无法跨语言（即 Java 序列化得到的二进制字节流无法被其他语言解析）、序列化之后的数据体积较大。因此在实际的应用中多采用 Protobuf 技术来代替 Netty 提供的部分编解码器。

Protobuf 是 Google 的开源项目（Protobuf 的官网文档链接为：https://developers.google.com/protocol-buffers/docs/proto），全称是 Google Protocol Buffers，是一种轻便高效、结构化的数据序列化框架，Protobuf 格式的数据很适合做数据存储或者 RPC 数据交换格式。而且，Protobuf 支持跨平台、跨语言（即序列化之后的数据可被多种语言直接解析），且性能高、序列化之后得到的数据体积较小。

Protobuf 框架对多种语言提供了编程支持。例如在使用 Java 语言编程时，需要先按照 Protobuf 语法编写一个 POJO 对象的定义文件，然后利用 Protobuf 框架提供的代码生成工具将该定义文件转换为一个定义 POJO 对象的 Java 类。最后使用 Netty 为 Protobuf 协议提供的编解码器（例如 ProtobufEncoder 和 ProtobufDecoder）对该类的对象进行编解码处理即可。而且在这个过程中粘包拆包问题被透明地处理了。

下面给出一个应用案例，在该案例中，客户端和服务器之间通过 Netty 传递 Student 信息。

1）编写 Student.proto 文件

```proto
syntax = "proto3";
option java_outer_classname = "StudentPOJO";

// 结构化对象 Student，包含两个属性：id 和 name
message Student {
    int32 id = 1; // int32 是 protobuf 定义的数据类型；1 和 2 是属性 id，而非属性值
    string name = 2;
}

// 更多 proto 文件语法说明，请参考 Protobuf 的官网文档
```

2）使用代码生成工具执行

```
protoc.exe --java_out=. Student.proto
```

生成 StudentPOJO.java 文件如下。代码较为复杂，描述了对象序列化相关的细节。有兴趣的读者可以结合 Protobuf 框架定义的数据协议阅读一下，无兴趣的读者可以跳过。

```java
// Generated by the protocol buffer compiler.  DO NOT EDIT!
// source: Student.proto

public final class StudentPOJO {
  private StudentPOJO() {}
  public static void registerAllExtensions(
          com.google.protobuf.ExtensionRegistryLite registry) {
  }

  public static void registerAllExtensions(
          com.google.protobuf.ExtensionRegistry registry) {
    registerAllExtensions(
            (com.google.protobuf.ExtensionRegistryLite) registry);
  }
  public interface StudentOrBuilder extends
          // @@protoc_insertion_point(interface_extends:Student)
          com.google.protobuf.MessageOrBuilder {

    /**
     * <code>int32 id = 1;</code>
     */
    int getId();

    /**
     * <code>string name = 2;</code>
     */
    java.lang.String getName();
    /**
     * <code>string name = 2;</code>
     */
    com.google.protobuf.ByteString
    getNameBytes();
  }
  /**
   * <pre>
   * 结构化对象 Student，包含两个属性：id 和 name
   * </pre>
   *
   * Protobuf type {@code Student}
   */
  public  static final class Student extends
          com.google.protobuf.GeneratedMessageV3 implements
          // @@protoc_insertion_point(message_implements:Student)
          StudentOrBuilder {
    private static final long serialVersionUID = 0L;
    // Use Student.newBuilder() to construct.
    private Student(com.google.protobuf.GeneratedMessageV3.Builder<?> builder) {
      super(builder);
    }
    private Student() {
      id_ = 0;
      name_ = "";
    }

    @java.lang.Override
    public final com.google.protobuf.UnknownFieldSet
    getUnknownFields() {
      return this.unknownFields;
    }
    private Student(
            com.google.protobuf.CodedInputStream input,
            com.google.protobuf.ExtensionRegistryLite extensionRegistry)
            throws com.google.protobuf.InvalidProtocolBufferException {
      this();
      if (extensionRegistry == null) {
        throw new java.lang.NullPointerException();
      }
      int mutable_bitField0_ = 0;
      com.google.protobuf.UnknownFieldSet.Builder unknownFields =
              com.google.protobuf.UnknownFieldSet.newBuilder();
      try {
        boolean done = false;
        while (!done) {
          int tag = input.readTag();
          switch (tag) {
            case 0:
              done = true;
              break;
            case 8: {

              id_ = input.readInt32();
              break;
            }
            case 18: {
              java.lang.String s = input.readStringRequireUtf8();

              name_ = s;
              break;
            }
            default: {
              if (!parseUnknownFieldProto3(
                      input, unknownFields, extensionRegistry, tag)) {
                done = true;
              }
              break;
            }
          }
        }
      } catch (com.google.protobuf.InvalidProtocolBufferException e) {
        throw e.setUnfinishedMessage(this);
      } catch (java.io.IOException e) {
        throw new com.google.protobuf.InvalidProtocolBufferException(
                e).setUnfinishedMessage(this);
      } finally {
        this.unknownFields = unknownFields.build();
        makeExtensionsImmutable();
      }
    }
    public static final com.google.protobuf.Descriptors.Descriptor
    getDescriptor() {
      return StudentPOJO.internal_static_Student_descriptor;
    }

    @java.lang.Override
    protected com.google.protobuf.GeneratedMessageV3.FieldAccessorTable
    internalGetFieldAccessorTable() {
      return StudentPOJO.internal_static_Student_fieldAccessorTable
              .ensureFieldAccessorsInitialized(
                      StudentPOJO.Student.class, StudentPOJO.Student.Builder.class);
    }

    public static final int ID_FIELD_NUMBER = 1;
    private int id_;
    /**
     * <code>int32 id = 1;</code>
     */
    public int getId() {
      return id_;
    }

    public static final int NAME_FIELD_NUMBER = 2;
    private volatile java.lang.Object name_;
    /**
     * <code>string name = 2;</code>
     */
    public java.lang.String getName() {
      java.lang.Object ref = name_;
      if (ref instanceof java.lang.String) {
        return (java.lang.String) ref;
      } else {
        com.google.protobuf.ByteString bs =
                (com.google.protobuf.ByteString) ref;
        java.lang.String s = bs.toStringUtf8();
        name_ = s;
        return s;
      }
    }
    /**
     * <code>string name = 2;</code>
     */
    public com.google.protobuf.ByteString
    getNameBytes() {
      java.lang.Object ref = name_;
      if (ref instanceof java.lang.String) {
        com.google.protobuf.ByteString b =
                com.google.protobuf.ByteString.copyFromUtf8(
                        (java.lang.String) ref);
        name_ = b;
        return b;
      } else {
        return (com.google.protobuf.ByteString) ref;
      }
    }

    private byte memoizedIsInitialized = -1;
    @java.lang.Override
    public final boolean isInitialized() {
      byte isInitialized = memoizedIsInitialized;
      if (isInitialized == 1) return true;
      if (isInitialized == 0) return false;

      memoizedIsInitialized = 1;
      return true;
    }

    @java.lang.Override
    public void writeTo(com.google.protobuf.CodedOutputStream output)
            throws java.io.IOException {
      if (id_ != 0) {
        output.writeInt32(1, id_);
      }
      if (!getNameBytes().isEmpty()) {
        com.google.protobuf.GeneratedMessageV3.writeString(output, 2, name_);
      }
      unknownFields.writeTo(output);
    }

    @java.lang.Override
    public int getSerializedSize() {
      int size = memoizedSize;
      if (size != -1) return size;

      size = 0;
      if (id_ != 0) {
        size += com.google.protobuf.CodedOutputStream
                .computeInt32Size(1, id_);
      }
      if (!getNameBytes().isEmpty()) {
        size += com.google.protobuf.GeneratedMessageV3.computeStringSize(2, name_);
      }
      size += unknownFields.getSerializedSize();
      memoizedSize = size;
      return size;
    }

    @java.lang.Override
    public boolean equals(final java.lang.Object obj) {
      if (obj == this) {
        return true;
      }
      if (!(obj instanceof StudentPOJO.Student)) {
        return super.equals(obj);
      }
      StudentPOJO.Student other = (StudentPOJO.Student) obj;

      boolean result = true;
      result = result && (getId()
              == other.getId());
      result = result && getName()
              .equals(other.getName());
      result = result && unknownFields.equals(other.unknownFields);
      return result;
    }

    @java.lang.Override
    public int hashCode() {
      if (memoizedHashCode != 0) {
        return memoizedHashCode;
      }
      int hash = 41;
      hash = (19 * hash) + getDescriptor().hashCode();
      hash = (37 * hash) + ID_FIELD_NUMBER;
      hash = (53 * hash) + getId();
      hash = (37 * hash) + NAME_FIELD_NUMBER;
      hash = (53 * hash) + getName().hashCode();
      hash = (29 * hash) + unknownFields.hashCode();
      memoizedHashCode = hash;
      return hash;
    }

    public static StudentPOJO.Student parseFrom(
            java.nio.ByteBuffer data)
            throws com.google.protobuf.InvalidProtocolBufferException {
      return PARSER.parseFrom(data);
    }
    public static StudentPOJO.Student parseFrom(
            java.nio.ByteBuffer data,
            com.google.protobuf.ExtensionRegistryLite extensionRegistry)
            throws com.google.protobuf.InvalidProtocolBufferException {
      return PARSER.parseFrom(data, extensionRegistry);
    }
    public static StudentPOJO.Student parseFrom(
            com.google.protobuf.ByteString data)
            throws com.google.protobuf.InvalidProtocolBufferException {
      return PARSER.parseFrom(data);
    }
    public static StudentPOJO.Student parseFrom(
            com.google.protobuf.ByteString data,
            com.google.protobuf.ExtensionRegistryLite extensionRegistry)
            throws com.google.protobuf.InvalidProtocolBufferException {
      return PARSER.parseFrom(data, extensionRegistry);
    }
    public static StudentPOJO.Student parseFrom(byte[] data)
            throws com.google.protobuf.InvalidProtocolBufferException {
      return PARSER.parseFrom(data);
    }
    public static StudentPOJO.Student parseFrom(
            byte[] data,
            com.google.protobuf.ExtensionRegistryLite extensionRegistry)
            throws com.google.protobuf.InvalidProtocolBufferException {
      return PARSER.parseFrom(data, extensionRegistry);
    }
    public static StudentPOJO.Student parseFrom(java.io.InputStream input)
            throws java.io.IOException {
      return com.google.protobuf.GeneratedMessageV3
              .parseWithIOException(PARSER, input);
    }
    public static StudentPOJO.Student parseFrom(
            java.io.InputStream input,
            com.google.protobuf.ExtensionRegistryLite extensionRegistry)
            throws java.io.IOException {
      return com.google.protobuf.GeneratedMessageV3
              .parseWithIOException(PARSER, input, extensionRegistry);
    }
    public static StudentPOJO.Student parseDelimitedFrom(java.io.InputStream input)
            throws java.io.IOException {
      return com.google.protobuf.GeneratedMessageV3
              .parseDelimitedWithIOException(PARSER, input);
    }
    public static StudentPOJO.Student parseDelimitedFrom(
            java.io.InputStream input,
            com.google.protobuf.ExtensionRegistryLite extensionRegistry)
            throws java.io.IOException {
      return com.google.protobuf.GeneratedMessageV3
              .parseDelimitedWithIOException(PARSER, input, extensionRegistry);
    }
    public static StudentPOJO.Student parseFrom(
            com.google.protobuf.CodedInputStream input)
            throws java.io.IOException {
      return com.google.protobuf.GeneratedMessageV3
              .parseWithIOException(PARSER, input);
    }
    public static StudentPOJO.Student parseFrom(
            com.google.protobuf.CodedInputStream input,
            com.google.protobuf.ExtensionRegistryLite extensionRegistry)
            throws java.io.IOException {
      return com.google.protobuf.GeneratedMessageV3
              .parseWithIOException(PARSER, input, extensionRegistry);
    }

    @java.lang.Override
    public Builder newBuilderForType() { return newBuilder(); }
    public static Builder newBuilder() {
      return DEFAULT_INSTANCE.toBuilder();
    }
    public static Builder newBuilder(StudentPOJO.Student prototype) {
      return DEFAULT_INSTANCE.toBuilder().mergeFrom(prototype);
    }
    @java.lang.Override
    public Builder toBuilder() {
      return this == DEFAULT_INSTANCE
              ? new Builder() : new Builder().mergeFrom(this);
    }

    @java.lang.Override
    protected Builder newBuilderForType(
            com.google.protobuf.GeneratedMessageV3.BuilderParent parent) {
      Builder builder = new Builder(parent);
      return builder;
    }
    /**
     * <pre>
     * 结构化对象 Student，包含两个属性：id 和 name
     * </pre>
     *
     * Protobuf type {@code Student}
     */
    public static final class Builder extends
            com.google.protobuf.GeneratedMessageV3.Builder<Builder> implements
            // @@protoc_insertion_point(builder_implements:Student)
            StudentPOJO.StudentOrBuilder {
      public static final com.google.protobuf.Descriptors.Descriptor
      getDescriptor() {
        return StudentPOJO.internal_static_Student_descriptor;
      }

      @java.lang.Override
      protected com.google.protobuf.GeneratedMessageV3.FieldAccessorTable
      internalGetFieldAccessorTable() {
        return StudentPOJO.internal_static_Student_fieldAccessorTable
                .ensureFieldAccessorsInitialized(
                        StudentPOJO.Student.class, StudentPOJO.Student.Builder.class);
      }

      // Construct using StudentPOJO.Student.newBuilder()
      private Builder() {
        maybeForceBuilderInitialization();
      }

      private Builder(
              com.google.protobuf.GeneratedMessageV3.BuilderParent parent) {
        super(parent);
        maybeForceBuilderInitialization();
      }
      private void maybeForceBuilderInitialization() {
        if (com.google.protobuf.GeneratedMessageV3
                .alwaysUseFieldBuilders) {
        }
      }
      @java.lang.Override
      public Builder clear() {
        super.clear();
        id_ = 0;

        name_ = "";

        return this;
      }

      @java.lang.Override
      public com.google.protobuf.Descriptors.Descriptor
      getDescriptorForType() {
        return StudentPOJO.internal_static_Student_descriptor;
      }

      @java.lang.Override
      public StudentPOJO.Student getDefaultInstanceForType() {
        return StudentPOJO.Student.getDefaultInstance();
      }

      @java.lang.Override
      public StudentPOJO.Student build() {
        StudentPOJO.Student result = buildPartial();
        if (!result.isInitialized()) {
          throw newUninitializedMessageException(result);
        }
        return result;
      }

      @java.lang.Override
      public StudentPOJO.Student buildPartial() {
        StudentPOJO.Student result = new StudentPOJO.Student(this);
        result.id_ = id_;
        result.name_ = name_;
        onBuilt();
        return result;
      }

      @java.lang.Override
      public Builder clone() {
        return (Builder) super.clone();
      }
      @java.lang.Override
      public Builder setField(
              com.google.protobuf.Descriptors.FieldDescriptor field,
              java.lang.Object value) {
        return (Builder) super.setField(field, value);
      }
      @java.lang.Override
      public Builder clearField(
              com.google.protobuf.Descriptors.FieldDescriptor field) {
        return (Builder) super.clearField(field);
      }
      @java.lang.Override
      public Builder clearOneof(
              com.google.protobuf.Descriptors.OneofDescriptor oneof) {
        return (Builder) super.clearOneof(oneof);
      }
      @java.lang.Override
      public Builder setRepeatedField(
              com.google.protobuf.Descriptors.FieldDescriptor field,
              int index, java.lang.Object value) {
        return (Builder) super.setRepeatedField(field, index, value);
      }
      @java.lang.Override
      public Builder addRepeatedField(
              com.google.protobuf.Descriptors.FieldDescriptor field,
              java.lang.Object value) {
        return (Builder) super.addRepeatedField(field, value);
      }
      @java.lang.Override
      public Builder mergeFrom(com.google.protobuf.Message other) {
        if (other instanceof StudentPOJO.Student) {
          return mergeFrom((StudentPOJO.Student)other);
        } else {
          super.mergeFrom(other);
          return this;
        }
      }

      public Builder mergeFrom(StudentPOJO.Student other) {
        if (other == StudentPOJO.Student.getDefaultInstance()) return this;
        if (other.getId() != 0) {
          setId(other.getId());
        }
        if (!other.getName().isEmpty()) {
          name_ = other.name_;
          onChanged();
        }
        this.mergeUnknownFields(other.unknownFields);
        onChanged();
        return this;
      }

      @java.lang.Override
      public final boolean isInitialized() {
        return true;
      }

      @java.lang.Override
      public Builder mergeFrom(
              com.google.protobuf.CodedInputStream input,
              com.google.protobuf.ExtensionRegistryLite extensionRegistry)
              throws java.io.IOException {
        StudentPOJO.Student parsedMessage = null;
        try {
          parsedMessage = PARSER.parsePartialFrom(input, extensionRegistry);
        } catch (com.google.protobuf.InvalidProtocolBufferException e) {
          parsedMessage = (StudentPOJO.Student) e.getUnfinishedMessage();
          throw e.unwrapIOException();
        } finally {
          if (parsedMessage != null) {
            mergeFrom(parsedMessage);
          }
        }
        return this;
      }

      private int id_ ;
      /**
       * <code>int32 id = 1;</code>
       */
      public int getId() {
        return id_;
      }
      /**
       * <code>int32 id = 1;</code>
       */
      public Builder setId(int value) {

        id_ = value;
        onChanged();
        return this;
      }
      /**
       * <code>int32 id = 1;</code>
       */
      public Builder clearId() {

        id_ = 0;
        onChanged();
        return this;
      }

      private java.lang.Object name_ = "";
      /**
       * <code>string name = 2;</code>
       */
      public java.lang.String getName() {
        java.lang.Object ref = name_;
        if (!(ref instanceof java.lang.String)) {
          com.google.protobuf.ByteString bs =
                  (com.google.protobuf.ByteString) ref;
          java.lang.String s = bs.toStringUtf8();
          name_ = s;
          return s;
        } else {
          return (java.lang.String) ref;
        }
      }
      /**
       * <code>string name = 2;</code>
       */
      public com.google.protobuf.ByteString
      getNameBytes() {
        java.lang.Object ref = name_;
        if (ref instanceof String) {
          com.google.protobuf.ByteString b =
                  com.google.protobuf.ByteString.copyFromUtf8(
                          (java.lang.String) ref);
          name_ = b;
          return b;
        } else {
          return (com.google.protobuf.ByteString) ref;
        }
      }
      /**
       * <code>string name = 2;</code>
       */
      public Builder setName(
              java.lang.String value) {
        if (value == null) {
          throw new NullPointerException();
        }

        name_ = value;
        onChanged();
        return this;
      }
      /**
       * <code>string name = 2;</code>
       */
      public Builder clearName() {

        name_ = getDefaultInstance().getName();
        onChanged();
        return this;
      }
      /**
       * <code>string name = 2;</code>
       */
      public Builder setNameBytes(
              com.google.protobuf.ByteString value) {
        if (value == null) {
          throw new NullPointerException();
        }
        checkByteStringIsUtf8(value);

        name_ = value;
        onChanged();
        return this;
      }
      @java.lang.Override
      public final Builder setUnknownFields(
              final com.google.protobuf.UnknownFieldSet unknownFields) {
        return super.setUnknownFieldsProto3(unknownFields);
      }

      @java.lang.Override
      public final Builder mergeUnknownFields(
              final com.google.protobuf.UnknownFieldSet unknownFields) {
        return super.mergeUnknownFields(unknownFields);
      }


      // @@protoc_insertion_point(builder_scope:Student)
    }

    // @@protoc_insertion_point(class_scope:Student)
    private static final StudentPOJO.Student DEFAULT_INSTANCE;
    static {
      DEFAULT_INSTANCE = new StudentPOJO.Student();
    }

    public static StudentPOJO.Student getDefaultInstance() {
      return DEFAULT_INSTANCE;
    }

    private static final com.google.protobuf.Parser<Student>
            PARSER = new com.google.protobuf.AbstractParser<Student>() {
      @java.lang.Override
      public Student parsePartialFrom(
              com.google.protobuf.CodedInputStream input,
              com.google.protobuf.ExtensionRegistryLite extensionRegistry)
              throws com.google.protobuf.InvalidProtocolBufferException {
        return new Student(input, extensionRegistry);
      }
    };

    public static com.google.protobuf.Parser<Student> parser() {
      return PARSER;
    }

    @java.lang.Override
    public com.google.protobuf.Parser<Student> getParserForType() {
      return PARSER;
    }

    @java.lang.Override
    public StudentPOJO.Student getDefaultInstanceForType() {
      return DEFAULT_INSTANCE;
    }

  }

  private static final com.google.protobuf.Descriptors.Descriptor
          internal_static_Student_descriptor;
  private static final
  com.google.protobuf.GeneratedMessageV3.FieldAccessorTable
          internal_static_Student_fieldAccessorTable;

  public static com.google.protobuf.Descriptors.FileDescriptor
  getDescriptor() {
    return descriptor;
  }
  private static  com.google.protobuf.Descriptors.FileDescriptor
          descriptor;
  static {
    java.lang.String[] descriptorData = {
            "\n\rStudent.proto\"#\n\007Student\022\n\n\002id\030\001 \001(\005\022\014" +
                    "\n\004name\030\002 \001(\tB\rB\013StudentPOJOb\006proto3"
    };
    com.google.protobuf.Descriptors.FileDescriptor.InternalDescriptorAssigner assigner =
            new com.google.protobuf.Descriptors.FileDescriptor.    InternalDescriptorAssigner() {
              public com.google.protobuf.ExtensionRegistry assignDescriptors(
                      com.google.protobuf.Descriptors.FileDescriptor root) {
                descriptor = root;
                return null;
              }
            };
    com.google.protobuf.Descriptors.FileDescriptor
            .internalBuildGeneratedFileFrom(descriptorData,
                    new com.google.protobuf.Descriptors.FileDescriptor[] {
                    }, assigner);
    internal_static_Student_descriptor =
            getDescriptor().getMessageTypes().get(0);
    internal_static_Student_fieldAccessorTable = new
            com.google.protobuf.GeneratedMessageV3.FieldAccessorTable(
            internal_static_Student_descriptor,
            new java.lang.String[] { "Id", "Name", });
  }

  // @@protoc_insertion_point(outer_class_scope)
}
```

3）编写客户端代码，要注意编码器 Handler 配置为 ProtobufEncoder 的实例，以及使用 StudentPOJO 中的构建器来创建要发送的 StudentPOJO.Student 的实例

```java
package example;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.protobuf.ProtobufEncoder;

public class MyNtyClient {
    /**
     * 需要的依赖：
     * <dependency>
     * <groupId>io.netty</groupId>
     * <artifactId>netty-all</artifactId>
     * <version>4.1.54.Final</version>
     * </dependency>
     *
     * <dependency>
     * <groupId>com.google.protobuf</groupId>
     * <artifactId>protobuf-java</artifactId>
     * <version>3.6.1</version>
     * </dependency>
     */
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap
                .group(eventLoopGroup)
                .channel(NioSocketChannel.class)
                .handler(
                        new ChannelInitializer<SocketChannel>() {
                            @Override
                            protected void initChannel(
                                    SocketChannel socketChannel
                            ) {
                                socketChannel.pipeline().addLast(
                                        // 注意这里！Protobuf 编码器 Handler
                                        new ProtobufEncoder()
                                ).addLast(
                                        // 业务处理 Handler
                                        new NettyClientHandler()
                                );
                            }
                        }
                );

            System.out.println("client is ready...");
            ChannelFuture channelFuture = bootstrap.connect(
                    "127.0.0.1",
                    8080).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            eventLoopGroup.shutdownGracefully();
        }
    }

    static class NettyClientHandler 
        extends SimpleChannelInboundHandler<StudentPOJO.Student> {

        @Override
        public void channelActive(ChannelHandlerContext ctx)
                throws Exception {
            // 发送一个 Student 对象到服务器
            StudentPOJO.Student student = StudentPOJO.Student.newBuilder()
                    .setId(1000)
                    .setName("tony")
                    .build();

            ctx.writeAndFlush(student);
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
                throws Exception {
            // 关闭与客户端的 Socket 连接
            cause.printStackTrace();
            ctx.channel().close();
        }

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, StudentPOJO.Student msg)
                throws Exception {
			// TODO
        }
    }
}
```

4）编写服务器代码，要注意解码器 Handler 配置为 ProtobufDecoder 的实例，并指定对 StudentPOJO.Student 的实例进行解码

```java
package example;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.*;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.protobuf.ProtobufDecoder;

public class MyNtyServer {
    /**
     * 需要的依赖：
     * <dependency>
     * <groupId>io.netty</groupId>
     * <artifactId>netty-all</artifactId>
     * <version>4.1.54.Final</version>
     * </dependency>
     *
     * <dependency>
     * <groupId>com.google.protobuf</groupId>
     * <artifactId>protobuf-java</artifactId>
     * <version>3.6.1</version>
     * </dependency>
     */
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap
                .group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, 128)
                .childOption(ChannelOption.SO_KEEPALIVE, true)
                .childHandler(
                        new ChannelInitializer<SocketChannel>() {
                            @Override
                            protected void initChannel(
                                    SocketChannel socketChannel
                            ) {
                                socketChannel.pipeline().addLast(
                                        // 注意这里！Protobuf 解码器 Handler
                                        new ProtobufDecoder(
                                                // 指定对哪种对象进行解码
                                                StudentPOJO.Student.getDefaultInstance()
                                        )
                                ).addLast(
                                        // 业务处理 Handler
                                        new NettyServerHandler()
                                );
                            }
                        }
                );

            System.out.println("server is ready...");
            ChannelFuture channelFuture = bootstrap.bind(8080).sync();
            channelFuture.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    static class NettyServerHandler 
        extends SimpleChannelInboundHandler<StudentPOJO.Student> {

        @Override
        protected void channelRead0(ChannelHandlerContext ctx, StudentPOJO.Student msg)
                throws Exception {
            // 从客户端读取 Student 对象
            StudentPOJO.Student student = msg;
            System.out.println("student id: " + student.getId()
                    + ", name: " + student.getName());
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
                throws Exception {
            // 关闭与服务器端的 Socket 连接
            cause.printStackTrace();
            ctx.channel().close();
        }
    }
}
```

5）先后启动 Server 和 Client，可以看到服务器成功接收到了客户端发来的 Student 对象。

![](https://img-blog.csdnimg.cn/img_convert/895e572ba285b2f37eeb132c07715e12.png)

在本案例中，客户端和服务器均使用 Java 编写。实际上，只要双方使用的结构化对象 Student 均由 Student.proto 文件生成，使用任何语言实现客户端和服务器均可，这就是 Protobuf 的跨语言特性。

更多 Protobuf 的使用细节请参考官方文档。

## 结束语

承接前两篇文章对 Netty 工作原理的探讨，本文作为应用篇给出了使用 Netty 进行网络 IO 应用开发的一系列要点，包括使用自定义协议+编解码器解决 TCP 传输中的粘包拆包问题、加密数据传输、HTTP/HTTPS 应用开发、WebSocket 应用开发、UDP 应用开发、基于 Netty 实现 RPC。最后指出 Netty 提供的消息序列化机制存在问题以及替代方案——ProtoBuf。关于 Netty 使用的话题较多，受篇幅限制，难以深入探讨，本文仅仅作为读者学习使用 Netty 的入门参考。