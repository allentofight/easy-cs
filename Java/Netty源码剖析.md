本文是发表于码海（seaofcode）公众号的《Netty 三讲》的第二讲：**Netty 核心源码解析（部分）**，大纲如下：

- [前言](#--)
- [1. Netty 服务端启动过程源码剖析](#1-netty------------)
  * [1.1. 执行 new NioEventLoopGroup()时发生了什么](#11----new-nioeventloopgroup--------)
  * [1.2. 服务端引导类 ServerBootstrap 的创建与配置](#12--------serverbootstrap-------)
  * [1.3. 执行 ServerBootstrap.bind(PORT)时发了什么](#13----serverbootstrapbind-port------)
- [2. Netty 服务端接收客户端连接请求源码剖析](#2-netty-----------------)
  * [2.1. 监听 accept 事件，接受连接 & 创建一个新的 NioSocketChannel](#21----accept------------------niosocketchannel)
  * [2.2. 将新的 NioSocketChannel 注册到 workerGroup 上](#22-----niosocketchannel-----workergroup--)
  * [2.3. 监听 NioSocketChannel 上的 Read 事件](#23----niosocketchannel----read---)
- [3. ChannelPipeline 源码剖析](#3-channelpipeline-----)
- [4. ChannelHandler 源码剖析](#4-channelhandler-----)
- [5. ChannelHandlerContext 源码剖析](#5-channelhandlercontext-----)
- [6. ChannelPipeline、ChannelHandler、ChannelHandlerContext 三者的创建过程源码剖析](#6-channelpipeline-channelhandler-channelhandlercontext------------)
- [7. ChannelPipeline 调用 ChannelHandler 源码剖析](#7-channelpipeline----channelhandler-----)
- [8. NioEventLoop 源码剖析](#8-nioeventloop-----)
- [结束语](#---)

本文第 1、2 两节通过追踪 Netty 服务端启动过程和接收请求过程的源码，展示 Netty 中 ServerBootStrap、EventLoopGroup、EventLoop、ChannelPipeline、ChannelHandler、ChannelHandlerContext 这些核心组件的工作原理，后面几节又对部分组件的源码做了专项的梳理和总结，以帮助读者加深对 Netty 架构和核心组件的理解。

## 前言

在[上一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)中，我首先给出了构建 Netty 的基础——Java NIO 的介绍，然后给出了常用的构建服务器程序的 Reactor 模式，接着通过一种渐进式的描述方式逐渐展示出 Netty 的架构，最终给出了如下的 Netty 架构图（图片来源于网络）：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/104.png)

最后，通过结合案例来详解 Netty 中的 Handler、Pipeline、EventLoopGroup 等组件，帮助读者加深对 Netty 架构的认识。

本篇文章将讲解 Netty 的核心源码，由于篇幅有限，只能拣选部分核心代码进行讲解。抛砖引玉，帮助大家上手 Netty 源码分析。
画外音：这篇文章篇幅虽然长，但是没有晦涩的理论知识，完全是对[上一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)的再消化，我建议读者跟随文章一起 debug 源码。

由于 Netty 本身的设计具有一定的复杂性，因此，与上一篇文章的风格一样，有些知识点我会在文中反复解释，以降低文章的阅读难度。

本文使用的是 4.1.54 版本的 netty 源码。netty 源码包的总体结构如下，在 io.netty.example 中，官方给我们提供了很多的实例供我们参考。有项目实战需求的读者在了解了 Netty 的工作原理和常用 API 之后，可以参考这个包中的案例构建自己的网络 IO 程序。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/100.png)

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/101.png)

建议读者跟随文章动手追踪源码，一起开始吧！

## 1. Netty 服务端启动过程源码剖析

我们使用 io.netty.example 中的 echo 案例来作为我们分析源码的起点。创建一个 maven 项目，将源码包中的 echo 案例拷贝到项目的源码目录，在 pom 中引入 netty 依赖，即可启动 echo 中的 Server 和 Client 程序。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/102.png)

```xml
<!-- netty maven 依赖 -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.54.Final</version>
</dependency>
```

其中，echo server 的代码如下。有了[上一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)的基础，我们很容易就能看出这段代码的意图，相关说明我已经以注释的方式写在了代码里。

```java
/**
 * EchoServer 将原样返回 EchoClient 发来的任何消息
 */
public final class EchoServer {
    static final boolean SSL = System.getProperty("ssl") != null;
    static final int PORT = Integer.parseInt(
        System.getProperty("port", "8007")
    );

    public static void main(String[] args) throws Exception {
        // 通过 SslContext 构建安全套接字（Secure Socket），
        // 这是 Netty 提供的安全特性
        final SslContext sslCtx;
        if (SSL) {
            SelfSignedCertificate ssc = new SelfSignedCertificate();
            sslCtx = SslContextBuilder.forServer(
                ssc.certificate(), 
                ssc.privateKey()
            ).build();
        } else {
            sslCtx = null;
        }

        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        final EchoServerHandler serverHandler = new EchoServerHandler();
        try {
            ServerBootstrap b = new ServerBootstrap();
            // 设置线程组
            b.group(bossGroup, workerGroup)
             // 说明服务器端通道的实现类（便于 Netty 做反射处理）
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             // 对服务端的 NioServerSocketChannel 添加 Handler
             // LoggingHandler 是 netty 内置的一种 ChannelDuplexHandler，
             // 既可以处理出站事件，又可以处理入站事件，即 LoggingHandler
             // 既记录出站日志又记录入站日志。
             .handler(new LoggingHandler(LogLevel.INFO))
             // 对服务端接收到的、与客户端之间建立的 SocketChannel 添加 Handler
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         // sslCtx.newHandler(ch.alloc())对传输的内容
                         // 做安全加密处理
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     // 如果需要的话，可以用 LoggingHandler 记录与客户端之
                     // 间的通信日志
                     // p.addLast(new LoggingHandler(LogLevel.INFO));
                     
                     // serverHandler 用来实现 echo
                     p.addLast(serverHandler);
                 }
             });

            // 启动服务器
            ChannelFuture f = b.bind(PORT).sync();

            // 等待服务端 NioServerSocketChannel 关闭
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

其中 ServerHandler 的代码为：

```java
/**
 * ServerHandler 用来实现 echo（回声，即原样返回 EchoClient 发来的任何消息）
 */
@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
    /**
     * 当通道有数据可读时执行
     *
     * @param ctx 上下文对象，可以从中取得相关联的 Pipeline、Channel 等
     * @param msg 客户端发送的数据
     * @throws Exception
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // 原样写回 EchoClient 发来的任何消息
        ctx.write(msg);
    }

    /**
    * 上面 channelRead()执行完成后，触发本函数的执行
    */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        // 出现异常的时候，关闭当前 SocketChannel
        cause.printStackTrace();
        ctx.close();
    }
}
```

它是一个 ChannelInboundHandler，用来处理入站事件，即下图红色部分，关于出站和入站，[上一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)也做了详细讲解，不再赘述。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/103.png)

客户端的代码不再给出，读者可以到 io.netty.example.echo 下面去查看。下面我们来分析，这个 EchoServer 启动的时候到底发生了什么。

### 1.1. 执行 new NioEventLoopGroup()时发生了什么

我们先分析下面这行代码执行过程中的内幕（创建 bossGroup 的过程同理）：

```java
EventLoopGroup workerGroup = new NioEventLoopGroup();
```

> **“**
>
> [上一篇](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)文章对 bossGroup 和 workerGroup 的解释是：
>
> bossGroup 和 workerGroup 是整个 Netty 的核心对象，我在上一篇文章讲解了服务器构建的主从 Reactor 多线程模式（也叫做服务器的 1+M+N 模式），其中的主 Reactor 就是这里的 bossGroup 里面的 EventLoop、从 Reactor 就这这里的 workerGroup 里面的 EventLoop。
>
> bossGroup 用于接收 TCP 连接请求，建立连接后会把连接转接给 workerGroup 来处理连接上的后续请求，比如**读取请求数据-->解码请求数据-->进行业务处理-->编码响应数据-->发送响应数据**这一整套流程。
>
> EventLoopGroup 是一个线程组，其中的每一个线程都在循环执行着三件事情：
>
> - **select**：轮训注册在其中的 Selector 上的 Channel 的 IO 事件
> - **processSelectedKeys**：在对应的 Channel 上处理 IO 事件
> - **runAllTasks**：再去以此循环处理任务队列中的其他任务
>
> **”**

单点调试这两行代码，查看 workerGroup 的“成分”，可以看到它包含 16 个 NioEventLoop，每个 NioEventLoop 里面有选择器、任务队列、执行器等等：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/106.png)

NioEventLoop 的继承关系如下图所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/110.png)

NioEventLoop 的继承关系非常复杂，因此它的功能也非常丰富。

我们接下来看下 NioEventLoop 中的选择器、任务队列、执行器等成分是从哪来的。

如下图所示，我们逐层追踪第二行代码 EventLoopGroup workerGroup = new NioEventLoopGroup()的底层调用，直至追踪到红色框圈住的构造方法：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/105.png)

红色框圈住的构造方法的源码为：

```java
/**
 * Create a new instance.
 *
 * @param nThreads          the number of threads that will be used by this instance.
 * @param executor          the Executor to use, or {@code null} if the default should 
 *                          be used.
 * @param chooserFactory    the {@link EventExecutorChooserFactory} to use.
 * @param args              arguments which will passed to each 
 *                          {@link #newChild(Executor, Object...)} call
 */
protected MultithreadEventExecutorGroup(
    int nThreads, 
    Executor executor, 
    EventExecutorChooserFactory chooserFactory, 
    Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(
            String.format("nThreads: %d (expected: > 0)", nThreads)
        );
    }

    // 这里的 ThreadPerTaskExecutor 实例是下文用于创建 EventExecutor 实例的参数
    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        // ThreadPerTaskExecutor 的源代码如下，它的功能是从线程工厂中获取线程来执行 command
        
        //public final class ThreadPerTaskExecutor implements Executor {
        //    private final ThreadFactory threadFactory;
        //
        //    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        //        this.threadFactory 
        //        = ObjectUtil.checkNotNull(threadFactory, "threadFactory");
        //    }
        //
        //    @Override
        //    public void execute(Runnable command) {
        //        threadFactory.newThread(command).start();
        //    }
        //}
    }

    // 这里定义了一个容量为 16 的 EventExecutor 的数组
    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            // 往 EventExecutor 数组中添加元素
            children[i] = newChild(executor, args);
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException(
                "failed to create a child event loop", 
                e
            );
        } finally {
            // 添加元素失败，则 shutdown 每一个 EventExecutor
            if (!success) {
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) {
                            e.awaitTermination(
                                Integer.MAX_VALUE, 
                                TimeUnit.SECONDS
                            );
                        }
                    } catch (InterruptedException interrupted) {
                        // Let the caller handle the interruption.
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }

    // chooser 的作用是为了实现 next()方法，即从 group 中挑选
    // 一个 NioEventLoop 来处理连接上 IO 事件的方法
    chooser = chooserFactory.newChooser(children);

    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        // EventExecutor 的终止事件回调方法
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                // 通过本类中定义的 Promise 属性的.setSuccess()方法设置结果，
                // 所有的监听者可以拿到该结果
                terminationFuture.setSuccess(null);
            }
        }
    };

    for (EventExecutor e: children) {
        // 为每一个 EventExecutor 添加终止事件监听器
        e.terminationFuture().addListener(terminationListener);
    }

    Set<EventExecutor> childrenSet 
        = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren 
        = Collections.unmodifiableSet(childrenSet);
}
```

可以从这段源码看到，children 里面的 16 个 NioEventLoop（其父类型为 EventExecutor）是通过如下的代码放入 workerGroup 里面去的：

```java
// 这里定义了一个容量为 16 的 EventExecutor 的数组
children = new EventExecutor[nThreads];

for (int i = 0; i < nThreads; i ++) {
    ......
    // 往 EventExecutor 数组中添加元素
    children[i] = newChild(executor, args);
    ......
```

这里的 newChild()方法包含了构建每一个 NioEventLoop 的细节，我们追踪进去：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/108.png)

可以看到，newChild()调用了 NioEventLoop 的构造函数来构建每一个 NioEventLoop 实例。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/116.png)

调用 NioEventLoop 的构造函数的时候，传入的参数 parent 为上一层调用者，executor 为 ThreadPerTaskExecutor 的实例，上文的代码注释已经讲明了其来源和功能（到此为止，已经讲明了 NioEventLoop 中的执行器的出处和用途了）：

```java
// 这里的 ThreadPerTaskExecutor 实例是下文用于创建 EventExecutor 实例的参数
if (executor == null) {
    executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
    // ThreadPerTaskExecutor 的源代码如下，它的功能是从线程工厂中获取线程来执行 command
    
    //public final class ThreadPerTaskExecutor implements Executor {
    //    private final ThreadFactory threadFactory;
    //
    //    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
    //        this.threadFactory 
    //        = ObjectUtil.checkNotNull(threadFactory, "threadFactory");
    //    }
    //
    //    @Override
    //    public void execute(Runnable command) {
    //        threadFactory.newThread(command).start();
    //    }
    //}
}
```

其余三个参数 selectorProvider、strategy、rejectedExecutionHandler 的来源分别如下三张图所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/107.png)

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/111.png)

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/112.png)

这三个参数的功能是用来创建 NioEventLoop 中的选择器和任务队列，下面具体来看。

NioEventLoop 的构造方法中有一个 openSelector()，它就完成了选择器（多路复用器）的初始化（对代码的解释写在了下面的代码注解中了）。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/109.png)

```java
/**
 * 
 */
private SelectorTuple openSelector() {
    final Selector unwrappedSelector;
    try {
        // 通过往下追踪发现 provider.openSelector()最终调用了
        // WindowsSelectorImpl 类的构造方法构造出一个 Selector，
        // 因此 unwrappedSelector 是 WindowsSelectorImpl 的实例
        unwrappedSelector = provider.openSelector();
    } catch (IOException e) {
        throw new ChannelException("failed to open a new selector", e);
    }

    // Netty 对 NIO 的 Selector 的 selectedKeys 进行了优化，用户可以
    // 通过 io.netty.noKeySetOptimization 开关决定是否启用该优化
    // 项
    //
    // 常量 DISABLE_KEY_SET_OPTIMIZATION = 
    //   SystemPropertyUtil.getBoolean(
    //      "io.netty.noKeySetOptimization",
    //      false
    //   );
    if (DISABLE_KEY_SET_OPTIMIZATION) {
        // 若没有开启 selectedKeys 优化，直接返回
        return new SelectorTuple(unwrappedSelector);
        
        // 上面的 SelectorTuple 构造函数源代码为：
        // SelectorTuple(Selector unwrappedSelector) {
        //     this.unwrappedSelector = unwrappedSelector;
        //     this.selector = unwrappedSelector;
        // }
    }

    // 若开启 selectedKeys 优化，需要通过反射的方式从 Selector 实例中
    // 获取 selectedKeys 和 publicSelectedKeys，将上述两个成员变量置
    // 为可写，然后通过反射的方式使用 Netty 构造的 selectedKeys 包装类
    // selectedKeySet 将原 JDK 的 selectedKeys 替换掉。
    
    // 以上这段表述就是后面的代码的内容
    
    Object maybeSelectorImplClass = AccessController.doPrivileged(
        new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    return Class.forName(
                            "sun.nio.ch.SelectorImpl",
                            false,
                            PlatformDependent.getSystemClassLoader()
                    );
                } catch (Throwable cause) {
                    return cause;
                }
            }
        }
    );

    if (!(maybeSelectorImplClass instanceof Class) ||
        // ensure the current selector implementation is what we can instrument.
        !((Class<?>) maybeSelectorImplClass)
        .isAssignableFrom(unwrappedSelector.getClass())
       ) 
    {
        if (maybeSelectorImplClass instanceof Throwable) {
            Throwable t = (Throwable) maybeSelectorImplClass;
            logger.trace(
                "failed to instrument a special java.util.Set into: {}", 
                unwrappedSelector, 
                t
            );
        }
        return new SelectorTuple(unwrappedSelector);
    }

    final Class<?> selectorImplClass 
        = (Class<?>) maybeSelectorImplClass;
    final SelectedSelectionKeySet selectedKeySet 
        = new SelectedSelectionKeySet();

    Object maybeException = AccessController.doPrivileged(
        new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                try {
                    Field selectedKeysField 
                        = selectorImplClass.getDeclaredField("selectedKeys");
                    Field publicSelectedKeysField 
                        = selectorImplClass.getDeclaredField("publicSelectedKeys");

                    if (PlatformDependent.javaVersion() >= 9 
                        && PlatformDependent.hasUnsafe()) {
                        // Let us try to use sun.misc.Unsafe to replace 
                        // the SelectionKeySet.
                        // This allows us to also do this in Java9+ without 
                        // any extra flags.
                        long selectedKeysFieldOffset 
                            = PlatformDependent.objectFieldOffset(selectedKeysField);
                        long publicSelectedKeysFieldOffset 
                            = PlatformDependent.objectFieldOffset(
                                  publicSelectedKeysField
                              );

                        if (selectedKeysFieldOffset != -1 
                            && publicSelectedKeysFieldOffset != -1) {
                            PlatformDependent.putObject(
                                unwrappedSelector, 
                                selectedKeysFieldOffset, 
                                selectedKeySet
                            );
                            PlatformDependent.putObject(
                                unwrappedSelector, 
                                publicSelectedKeysFieldOffset, 
                                selectedKeySet
                            );
                            return null;
                        }
                        // We could not retrieve the offset, lets try reflection 
                        // as last-resort.
                    }

                    Throwable cause = ReflectionUtil.trySetAccessible(
                        selectedKeysField, 
                        true
                    );
                    if (cause != null) {
                        return cause;
                    }
                    cause = ReflectionUtil.trySetAccessible(
                        publicSelectedKeysField, 
                        true
                    );
                    if (cause != null) {
                        return cause;
                    }

                    selectedKeysField.set(unwrappedSelector, selectedKeySet);
                    publicSelectedKeysField.set(unwrappedSelector, selectedKeySet);
                    return null;
                } catch (NoSuchFieldException e) {
                    return e;
                } catch (IllegalAccessException e) {
                    return e;
                }
            }
        }
    );

    if (maybeException instanceof Exception) {
        selectedKeys = null;
        Exception e = (Exception) maybeException;
        logger.trace(
            "failed to instrument a special java.util.Set into: {}", 
            unwrappedSelector, 
            e
        );
        return new SelectorTuple(unwrappedSelector);
    }
    selectedKeys = selectedKeySet;
    logger.trace(
        "instrumented a special java.util.Set into: {}", 
        unwrappedSelector
    );
    return new SelectorTuple(
        unwrappedSelector,
        new SelectedSelectionKeySetSelector(
            unwrappedSelector, 
            selectedKeySet
        )
    );
}
```

至此，已经讲清楚了 NioEventLoop 中选择器的由来了。

我们接着来追踪 NioEventLoop 中的任务队列从哪来的。回到 NioEventLoop 的构造器中来看：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/113.png)

红色圈住的三个参数就是用来构造任务队列的，其中 newTaskQueue()根据参数 queueFactory 产生一个 Queue<Runnable\>的实例，我们进到这个 super 方法，追踪下去：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/114.png)

再进入下一个 super，看到 newTaskQueue()根据参数 queueFactory 产生的 Queue<Runnable\>实例最终被赋值给了 SingleThreadEventExecutor 的 taskQueue 属性，taskQueue 是 SingleThreadEventExecutor 中的任务队列，而 NioEventLoop 又继承于 SingleThreadEventExecutor，因此 NioEventLoop 也就具有这个任务队列了。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/115.png)

同理，NioEventLoop 中的定时任务队列 scheduledTaskQueue 也是这么得到的：AbstractScheduledEventExecutor 包含 scheduledTaskQueue 属性，NioEventLoop 又继承于 AbstractScheduledEventExecutor，构造 NioEventLoop 的时候初始化这个 scheduledTaskQueue，因此 NioEventLoop 就有了定时任务队列。

至此，已经讲清楚了 NioEventLoop 中任务队列的由来了。

回到问题的起点，我们是要探究代码 EventLoopGroup workerGroup = new NioEventLoopGroup()的执行内幕，到此为止，答案已经清晰了，总结如下：

1）NioEventLoopGroup 的无参数构造函数会调用 NioEventLoopGroup 的有参数构造函数，最终把参数

- nThreads=16

- executor=null

- chooserFactory=DefaultEventExecutorChooserFactory.INSTANCE

- selectorProvider=SelectorProvider.provider()

- selectStrategyFactory=DefaultSelectStrategyFactory.INSTANCE

- rejectedExecutionHandler=RejectedExecutionHandlers.reject()

  传递给父类 MultithreadEventLoopGroup 的有参数构造函数。

2）父类 MultithreadEventLoopGroup 的有参数构造函数创建一个 NioEventLoop 的容器 children = new EventExecutor[nThreads]，并构建出 16 个 NioEventLoop 的实例放入其中。

3）构建每一个 NioEventLoop 调用的是 children[i] = newChild(executor, args)。

4）newChild()方法最终调用了 NioEventLoop 的构造函数，初始化其中的选择器、任务队列、执行器等成分。

本节只详述了 NioEventLoop 中选择器、任务队列、执行器三个成分的用途和由来，对于其他成分，读者可按照本节的代码追踪思路继续探究。

### 1.2. 服务端引导类 ServerBootstrap 的创建与配置

本节我们一起看下服务端启动类 ServerBootstrap 的创建与配置代码背后的逻辑。

```java
ServerBootstrap b = new ServerBootstrap();
// 设置线程组
b.group(bossGroup, workerGroup)
        // 说明服务器端通道的实现类（便于 Netty 做反射处理）
        .channel(NioServerSocketChannel.class)
        .option(ChannelOption.SO_BACKLOG, 100)
        // 对服务端的 NioServerSocketChannel 添加 Handler
        // LoggingHandler 是 netty 内置的一种 ChannelDuplexHandler，
        // 既可以处理出站事件，又可以处理入站事件，即 LoggingHandler
        // 既记录出站日志又记录入站日志。
        .handler(new LoggingHandler(LogLevel.INFO))
        // 对服务端接收到的、与客户端之间建立的 SocketChannel 添加 Handler
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {
                ChannelPipeline p = ch.pipeline();
                if (sslCtx != null) {
                    // sslCtx.newHandler(ch.alloc())对传输的内容
                    // 做安全加密处理
                    p.addLast(sslCtx.newHandler(ch.alloc()));
                }
                // 如果需要的话，可以用 LoggingHandler 记录与客户端之
                // 间的通信日志
                // p.addLast(new LoggingHandler(LogLevel.INFO));

                // serverHandler 用来实现 echo
                p.addLast(serverHandler);
            }
        });
```

ServerBootstrap 提供了一些列的链式配置方法，具体而言就是 ServerBootstrap 对象的一个配置方法（比如.group()）处理完配置参数之后，会将当前 ServerBootstrap 对象返回，这样就能紧随其后继续调用该对象的其他配置方法（比如.channel()）。这是面向对象语言中常见的一种编程模式。

我们按照下图所示断住代码，此时 ServerBootstrap 刚刚被创建，且未进行设置。此时这个对象 b 的成分如图右部分所示，它包含一个 ServerBootstrapConfig 对象，而这个对象又引用了 ServerBootstrap 对象，因此两个是互相引用、互相包含的关系。此外，对象 b 还包含了 childGroup、childHandler、group、handler 等成分，目前这些成分都为 null，后面进行的各种设置就是为这些成分赋值。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/117.png)

> ServerBootstrap 和 Bootstrap 一样，都继承于抽象类 AbstractBootstrap。
>
> ![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/125.png)
>
> 因此两者具备很多相同的属性和 API，例如 group、channelFactory、localAddress、options、attrs、handler、channel()、channelFactory()、register()、bind()等等。

第一项设置.group(bossGroup, workerGroup)的作用是把 bossGroup 和 workerGroup 两个参数赋值给 ServerBootstrap 的成员变量 group（从父类 AbstractBootstrap 继承而来）和 childGroup。如下图所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/118.png)

这样的话，ServerBootstrap 实例中的 group 和 childGroup 成分就有了值。

第二项配置.channel(NioServerSocketChannel.class)的作用是通过反射机制给当前 ServerBootstrap 中的 channelFactory 属性（从父类 AbstractBootstrap 继承而来）赋值。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/120.png)

而服务端的 NioServerSocketChannel 实例就是通过这个 channelFactory 创建的，不过现在还没有开始创建，要等到后面调用.bind()的时候才会创建，创建 NioServerSocketChannel 实例的源码追踪如下图所示。可以看到，最终在 initAndRegister()方法里面，NioServerSocketChannel 的实例 channel 被创建了出来。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/121.png)

> 注：也正是在上图的最后一个红色框圈住的代码处，NioServerSocketChannel 的实例被注册到 bossGroup 中 EventLoop 中的 Selector 上（ops: 0 为读事件的代码），读者可以追踪下去，最终到达如下的代码：
>
> ![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/130.png)

channel 被创建出来之后，紧接着后面的 init(channel)方法对该 channel 进行的初始化：

```java
@Override
void init(Channel channel) {
    // EchoServer 中通过.option()设置的 TCP 参数就在这里应用
    // setChannelOptions 方法的定义为：
    // static void setChannelOptions(
    //     Channel channel, 
    //     Map.Entry<ChannelOption<?>, 
    //     Object>[] options, 
    //     InternalLogger logger) {
    //     for (Map.Entry<ChannelOption<?>, Object> e: options) {
    //         setChannelOption(channel, e.getKey(), e.getValue(), logger);
    //     }
    // }
    setChannelOptions(channel, newOptionsArray(), logger);
    // EchoServer 中通过.attr()设置的附加属性就在这里应用
    // （实际上 EchoServer 并没有调用.attr()方法）
    // setAttributes 方法的定义为：
    // static void setAttributes(
    //     Channel channel, 
    //     Map.Entry<AttributeKey<?>, 
    //     Object>[] attrs) {
    //     for (Map.Entry<AttributeKey<?>, Object> e: attrs) {
    //         @SuppressWarnings("unchecked")
    //         AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
    //         channel.attr(key).set(e.getValue());
    //     }
    // }
    setAttributes(channel, attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY));

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions 
            = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs 
        = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, 
                        currentChildGroup, 
                        currentChildHandler, 
                        currentChildOptions, 
                        currentChildAttrs
                    ));
                }
            });
        }
    });
}
```

这里还有个问题：channel.pipelile()返回的 Pipeline 实例从哪来？其实 channelFactory.newChannel()通过反射调用了 NioServerSocketChannel 的无参数构造方法，追踪下去，这个无参数构造方法最终调用了父类 AbstractChannel 的构造方法，如下图所示。正是在这个 AbstractChannel 的构造方法中创建了 channel.pipelile()所返回的 Pipeline 实例。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/124.png)

第三项配置.option(ChannelOption.SO_BACKLOG, 100)的作用是将可选项放入一个 ChannelOption 集合中，如下图所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/122.png)

关于放进集合中的可选项配置<ChannelOption.SO_BACKLOG, 100>是怎么发挥作用的，后文详述。

第四项配置.handler(new LoggingHandler(LogLevel.INFO))的作用是将 Netty 提供的一个日志记录 Handler 赋值给 ServerBootstrap 实例的 handler 属性（从父类 AbstractBootstrap 继承而来）。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/123.png)

这个 handler 最终在 ServerBootstrap.init()方法中被放入 NioServerSocketChannel 实例的 pipeline 中，如下所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/126.png)

```java
/**
 * ServerBootstrap.init()方法，它在 channel = channelFactory.newChannel()
 * 之后被执行，用于初始化这个 channel
 */
@Override
void init(Channel channel) {
    setChannelOptions(channel, newOptionsArray(), logger);
    setAttributes(
        channel, 
        attrs0().entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY)
    );

    ChannelPipeline p = channel.pipeline();

    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    synchronized (childOptions) {
        currentChildOptions 
            = childOptions.entrySet().toArray(EMPTY_OPTION_ARRAY);
    }
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs 
        = childAttrs.entrySet().toArray(EMPTY_ATTRIBUTE_ARRAY);

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) {
            // 获取 NioServerSocketChannel 实例的 pipeline
            final ChannelPipeline pipeline = ch.pipeline();
            
            // 注意这里！！！
            // 这里的 config.handler()就是那个 new LoggingHandler(LogLevel.INFO)
            ChannelHandler handler = config.handler();
            if (handler != null) {
                // 将这个 handler 添加到 NioServerSocketChannel 实例的 pipeline 中
                pipeline.addLast(handler);
            }

            // 异步执行向 pipeline 添加 ServerBootstrapAcceptor 的步骤
            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    // ServerBootstrapAcceptor 是一个 ChannelInboundHandler
                    pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, 
                        currentChildGroup, 
                        currentChildHandler, 
                        currentChildOptions, 
                        currentChildAttrs
                    ));
                }
            });
        }
    });
}
```

与第四项配置同理，第五项配置.childHandler(...)的作用是为接收客户端连接请求产生的 NioSocketChannel 实例的 pipeline 添加两个 Handler，分别是做 SSL 加密处理的 Handler，以及一个我们自定义的 Handler：

```java
.childHandler(new ChannelInitializer<SocketChannel>() {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline p = ch.pipeline();
        if (sslCtx != null) {
            // sslCtx.newHandler(ch.alloc())对传输的内容
            // 做安全加密处理
            p.addLast(sslCtx.newHandler(ch.alloc()));
        }
        // 如果需要的话，可以用 LoggingHandler 记录与客户端之
        // 间的通信日志
        // p.addLast(new LoggingHandler(LogLevel.INFO));

        // serverHandler 用来实现 echo
        p.addLast(serverHandler);
    }
});
```

p.addLast(serverHandler)语句调用了 Pipeline 的 addLast 方法向 Pipeline 中的双向链表添加 ChannelHandlerContext 元素：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/128.png)

至此，对 ServerBootstrap 实例的创建与配置背后的 Netty 源码运作细节已经讲清楚了。总结如下：

1）.group(bossGroup, workerGroup)把 bossGroup 和 workerGroup 两个参数赋值给 ServerBootstrap 的成员变量 group（从父类 AbstractBootstrap 继承而来）和 childGroup。

2）.channel(NioServerSocketChannel.class)通过反射机制给当前 ServerBootstrap 中的 channelFactory 属性（从父类 AbstractBootstrap 继承而来）赋值。在调用.bind()的时候 channelFactory 会创建 NioServerSocketChannel 的实例。

3）.option(ChannelOption.SO_BACKLOG, 100)将可选项放入一个 ChannelOption 集合中。

4）.handler(new LoggingHandler(LogLevel.INFO))将 Netty 提供的一个日志记录 Handler 赋值给 ServerBootstrap 实例的 handler 属性（从父类 AbstractBootstrap 继承而来）。这个 Handler 最终在 ServerBootstrap.init()方法中被放入 NioServerSocketChannel 实例的 pipeline 中。

5）.childHandler(...)的作用是为接收客户端连接请求产生的 NioSocketChannel 实例的 pipeline 添加两个 Handler，分别是做 SSL 加密处理的 Handler，以及一个我们自定义的 Handler。

### 1.3. 执行 ServerBootstrap.bind(PORT)时发了什么

EchoServer 中启动服务器的代码 b.bind(PORT)调用了 AbstractBootstrap 中的 doBind()方法。该方法的源码如下（对代码的解说写在了注释中）：

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    // 初始化 NioServerSocketChannel 的实例，并且将其注册到
    // bossGroup 中的 EvenLoop 中的 Selector 中，initAndRegister()
    // 方法中有如下两句关键代码，分别完成 NioServerSocketChannel
    // 实例的初始化和注册：
    // (1) channel = channelFactory.newChannel();
    // (2) ChannelFuture regFuture = config().group().register(channel);
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        // 若异步过程 initAndRegister()已经执行完毕，则进入该分支
        ChannelPromise promise = channel.newPromise();
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        // 若异步过程 initAndRegister()还未执行完毕，则进入该分支
        final PendingRegistrationPromise promise 
            = new PendingRegistrationPromise(channel);
        
        regFuture.addListener(new ChannelFutureListener() {
            // 监听 regFuture 的完成事件，完成之后再调用
            // doBind0(regFuture, channel, localAddress, promise);
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    // Registration on the EventLoop failed so fail 
                    // the ChannelPromise directly to not cause an
                    // IllegalStateException once we try to access 
                    // the EventLoop of the Channel.
                    promise.setFailure(cause);
                } else {
                    // Registration was successful, so set the correct executor to use.
                    // See https://github.com/netty/netty/issues/2586
                    promise.registered();
                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```

对上面代码中的 doBind0(regFuture, channel, localAddress, promise)继续追踪，发现 doBind0(regFuture, channel, localAddress, promise)接着调用了 channel 的 bind()方法，最终调用了一个 Native 方法把.bind(PORT)最终托管给了 JVM，然后 JVM 进行系统调用。追踪过程如下：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/127.png)

在 NioServerSocketChannel 中的 javaChannel().bind(localAddress, config.getBacklog())调用底层 JDK 接口完成端口绑定和监听之后，继续追踪，会发现代码进入到了 NioEventLoop 中 run 方法的死循环里：

```java
@Override
protected void run() {
    int selectCnt = 0;
    for (;;) {
        try {
            int strategy;
            try {
                strategy = selectStrategy
                    .calculateStrategy(selectNowSupplier, hasTasks());
                switch (strategy) {
                case SelectStrategy.CONTINUE:
                    continue;

                case SelectStrategy.BUSY_WAIT:
                    // fall-through to SELECT since the busy-wait 
                    // is not supported with NIO
                case SelectStrategy.SELECT:
                    long curDeadlineNanos = nextScheduledTaskDeadlineNanos();
                    if (curDeadlineNanos == -1L) {
                        // nothing on the calendar
                        curDeadlineNanos = NONE;
                    }
                    nextWakeupNanos.set(curDeadlineNanos);
                    try {
                        if (!hasTasks()) {
                            strategy = select(curDeadlineNanos);
                        }
                    } finally {
                        // This update is just to help block unnecessary 
                        // selector wakeups so use of lazySet is ok 
                        // (no race condition)
                        nextWakeupNanos.lazySet(AWAKE);
                    }
                    // fall through
                default:
                }
            } catch (IOException e) {
                // If we receive an IOException here its because the 
                // Selector is messed up. Let's rebuild the selector 
                // and retry. https://github.com/netty/netty/issues/8566
                rebuildSelector0();
                selectCnt = 0;
                handleLoopException(e);
                continue;
            }

            selectCnt++;
            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            boolean ranTasks;
            if (ioRatio == 100) {
                try {
                    if (strategy > 0) {
                        processSelectedKeys();
                    }
                } finally {
                    // Ensure we always run tasks.
                    ranTasks = runAllTasks();
                }
            } else if (strategy > 0) {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime 
                        = System.nanoTime() - ioStartTime;
                    ranTasks 
                        = runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            } else {
                // This will run the minimum number of tasks
                ranTasks = runAllTasks(0); 
            }

            if (ranTasks || strategy > 0) {
                if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS 
                    && logger.isDebugEnabled()) {
                    logger.debug(
                        "Selector.select() returned prematurely {} " 
                        + "times in a row for Selector {}.",
                            selectCnt - 1, selector
                    );
                }
                selectCnt = 0;
            } else if (unexpectedSelectorWakeup(selectCnt)) { 
                // Unexpected wakeup (unusual case)
                selectCnt = 0;
            }
        } catch (CancelledKeyException e) {
            // Harmless exception - log anyway
            if (logger.isDebugEnabled()) {
                logger.debug(
                    CancelledKeyException.class.getSimpleName() 
                    + " raised by a Selector {} - JDK bug?",
                    selector, 
                    e
                );
            }
        } catch (Error e) {
            throw (Error) e;
        } catch (Throwable t) {
            handleLoopException(t);
        } finally {
            // Always handle shutdown even if the loop 
            // processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Error e) {
                throw (Error) e;
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
}
```

这段死循环就是在做下图（图片来源于网络）中红色框圈住的事情，这个过程我在上一篇文章（[点我查看上一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)）中已经做了详细的描述：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/129.png)

至此，对 ServerBootstrap 实例的.bind(PORT)背后的 Netty 源码运作细节已经讲清楚了。总结如下：

1）首先调用 AbstractBootstrap 中的 doBind()方法完成 NioServerSocketChannel 实例的初始化和注册。

2）然后调用 NioServerSocketChannel 实例的 bind()方法。

3）NioServerSocketChannel 实例的 bind()方法最终调用 sun.nio.ch.Net 中的 bind()和 listen()完成端口绑定和客户端连接监听。

4）sun.nio.ch.Net 中的 bind()和 listen()底层都是 JVM 进行的系统调用。

5）bind 完成后会进入 NioEventLoop 中的死循环，不断执行以下三个过程

- **select**：轮训注册在其中的 Selector 上的 Channel 的 IO 事件
- **processSelectedKeys**：在对应的 Channel 上处理 IO 事件
- **runAllTasks**：再去以此循环处理任务队列中的其他任务

## 2. Netty 服务端接收客户端连接请求源码剖析

在之前的服务器启动源码分析中，我们得知：服务器端的 NioServerSocketChannel 实例将自己注册到了 bossGroup 上（讲得更细一些，是 bossGroup 中 EventLoop 的 Selector 上），监听客户端连接，如下图所示。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/130.png)

Netty 服务端接收客户端连接请求的总体流程为：监听 Accept 事件，接受连接-->创建一个新的 NioSocketChannel-->将新的 NioSocketChannel 注册到 workerGroup 上-->监听 NioSocketChannel 上的 Read 事件。下面追踪代码来验证这一过程。

### 2.1. 监听 accept 事件，接受连接 & 创建一个新的 NioSocketChannel

之前说过，NioEventLoop 中的死循环，会不断执行以下三个过程

- **select**：轮训注册在其中的 Selector 上的 Channel 的 IO 事件
- **processSelectedKeys**：在对应的 Channel 上处理 IO 事件
- **runAllTasks**：再去以此循环处理任务队列中的其他任务

我们先以 Debug 模式启动 EchoServer，然后将断点放在 NioEventLoop 中 run 方法里面死循环代码块的 processSelectedKeys()语句上。再以 Run 模式启动 EchoClient。追踪服务端代码的执行，过程如下：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/131.png)

上图最后一个断点处调用的 AbstractNioMessageChannel$NioMessageUnsafe.read 方法的定义如下：

```java
/**
 * NioMessageUnsafe 是一个定义在 AbstractNioMessageChannel 中的内部类
 */
private final class NioMessageUnsafe extends AbstractNioUnsafe {

    // 可以看做存放请求数据的容器
    private final List<Object> readBuf 
        = new ArrayList<Object>();

    @Override
    public void read() {
        ......
    }
}

```

继续执行代码进入这个 read 方法，如下：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/132.png)

可以发现 read 方法里面通过 ServerSocketChannel 的 accept 方法创建了一个与客户端交互的 SocketChannel。

### 2.2. 将新的 NioSocketChannel 注册到 workerGroup 上

上一张图“**这里发生了什么？？？**”处留下了一个未解开的谜团，我们来看下这里面到底在干什么：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/133.png)

继续追踪上图的最后一个红色框圈住的 register 方法，最终过程如下所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/134.png)

不错，我们之前一直重复的“bossGroup 会将 NioServerSocketChannel 产生的 NioSocketChannel 注册到 workerGroup”就发生在这里！

### 2.3. 监听 NioSocketChannel 上的 Read 事件

“bossGroup 会将 NioServerSocketChannel 产生的 NioSocketChannel 注册到 workerGroup”这个过程完成之后，会触发 NioSocketChannel 中 doBeginRead 的执行（追到这一步的过程很漫长，此处不再给出过程），如下图所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/135.png)

这个过程完成了对 Read 事件的关注（上一篇文章中讲过，将 Channel 注册到 Selector 上的时候，会注册对哪些事件感兴趣，即关注哪些事件）。接下来 EventLoop 就可以对这个 NioSocketChannel 监听并处理 Read 事件了。

至此，Netty 服务端接收客户端连接请求的源码剖析已经讲清楚了。总结如下：

1）服务器端 bossGroup 中的 EventLoop 轮训 Accept 事件、获取事件后在 processSelectedKey()方法中调用 unsafe.read()方法，这个 unsafe 是内部类 io.netty.channel.nio.AbstractNioChannel.NioUnsafe 的实例，unsafe.read()方法由两个核心步骤组成：doReadMessages()和 pipeline.fireChannelRead()。

2）doReadMessages()用于创建 NioSocketChannel 对象，包装了 JDK 的 SocketChannel 对象，并且添加了 pipeline、unsafe、config 等成分。

3）pipeline.fireChannelRead()用于触发服务端 NioServerSocketChannel 的所有入站 Handler 的 channelRead()方法，在其中的一个类型为 ServerBootstrapAcceptor 的入站 Handler 的 channelRead()方法中将新创建的 NioSocketChannel 对象注册到 workerGroup 中的一个 EventLoop 上，该 EventLoop 开始监听 NioSocketChannel 中的读事件。

## 3. ChannelPipeline 源码剖析

这一节对 ChannelPipeline 的源码进行梳理和总结。

在梳理 ChannelPipeline 源码之前，我再唠叨一下 ChannelPipeline、ChannelHandler、ChannelHandlerContext 三者的关系，这在[上一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)中已经进行过详细的讲解：

1）每当 ServerSocketChannel 创建一个新的连接，就会创建一个 SocketChannel 对应目标客户端

2）每一个新创建的 SocketChannel 包含一个全新的 ChannelPipeline

3）每一个 ChannelPipeline 内部都含有一个由 ChannelHandlerContext 构成的双向链表

4）ChannelHandlerContext 都包装了一个 ChannelHandler

ChannelPipeline 的继承关系如下：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/136.png)

ChannelPipeline 本身是一个接口，它又继承了 ChannelInboundInvoker、ChannelOutboundInvoker、Iterable<Entry<String, ChannelHandler>>三个接口。这个继承关系很好理解，ChannelPipeline 要遍历其中的每一个 Handler 处理入站事件和出站事件，自然要继承 ChannelInboundInvoker、ChannelOutboundInvoker、Iterable<Entry<String, ChannelHandler>>三个接口。

它提供的接口有：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/137.png)

可以看到这些接口分这么几类：

- 元素操作类：以 get、add、remove、replace 等开头的
- Pipeline 里 Handler 责任链中 IO 事件处理方法触发类：以 fire、write 等开头的
- 其他

对于往 Pipeline 中添加 ChannelHandlerContext 元素这一过程，在前文 1.2 小节中已经进行了源码分析。而对于删除、修改和获取 Pipeline 中元素的过程，分析思路类似，本质就是在操作一个双向链表，因此本节不再赘述。本节着重分析 IO 事件在 Pipeline 中流转过程的源码。

在 ChannelPipeline 的源码的注释中，有这么一张字符画：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/138.png)

它表达意思和下面这幅图是一样的：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/103.png)

那就是：出站事件会依次流经 Pipeline 中每一个 ChannelHandlerContext，遇到 OutboundHandler 就被处理；入站事件会依次流经 Pipeline 中每一个 ChannelHandlerContext，遇到 InboundHandler 就被处理。

还以上文一直使用的 EchoServer/EchoClient 为例，以 Debug 模式启动 EchoServer，然后将断点放在 NioEventLoop 中 run 方法里面死循环代码块的 processSelectedKeys()语句上。再以 Run 模式启动 EchoClient。追踪服务端代码的执行直到 pipeline 露面，过程如下：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/139.png)

上图最后一个红框圈住的 fireChannelRead()方法表示触发 pipeline 中所有的能够处理该入站事件（Read 事件）的 HandlerContext 的 channelRead()方法。我们追进去（Step into）：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/140.png)

可以发现，确实是这样：循环查找 HandlerContext 双向链表中的元素，直到找到能够处理入站事件 Read 的 InboundHandler 类型的 HandlerHandlerContext，然后交由这个 HandlerHandlerContext 处理入站事件 Read。

## 4. ChannelHandler 源码剖析

这一节对 ChannelHandler 的源码进行梳理和总结。

通常，一个 Pipeline 中有多个 Handler，例如一个典型的服务器在每个管道中都会有协议解码器、协议编码器、业务处理程序，分别用来将接收到的二进制数据转换为 Java 对象，以及将要发送的 Java 转换为二进制数据，以及根据接收到的 Java 对象执行业务处理过程。那么每一个协议解码器、协议编码器、业务处理程序都可以定义成一个 Handler。

> 注：如果 Handler 里面的数据处理过程很快，可以放在当前 EventLoop 中处理，否则就要放入任务队列进行异步处理，或者开辟新的线程来处理。

ChannelHandler 是 Netty 中的一个顶级接口，它的子类和子接口有很多：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/141.png)

在这些子类和子接口中，有的是为了让使用者自定义入站/出站处理过程的接口（如 ChannelInboundHandler 和 ChannelOutboundHandler），有的是为使用者自定义入站/出站处理过程时提供便利的 Adapter，有的是 Netty 内置的用于实现网络协议的 Handler。

ChannelHandler 的源码很简单，如下：

```java
public interface ChannelHandler {

    /**
     * 回调函数，Handler 加入 Pipeline 后触发
     */
    void handlerAdded(ChannelHandlerContext ctx) throws Exception;

    /**
     * 回调函数，Handler 从 Pipeline 中移除后触发
     */
    void handlerRemoved(ChannelHandlerContext ctx) throws Exception;

    /**
     * 回调函数，Handler 处理 IO 事件时遇到异常后触发
     */
    @Deprecated
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
    
    ......
}
```

ChannelOutboundHandler 的源码也很简单，包含一些跟连接和写出数据相关的方法，如下：

```java
public interface ChannelOutboundHandler extends ChannelHandler {
    /**
     * 以下方法均在处理对应的事件（bind、connect、disconnect、close）时被调用
     */
    void bind(
        ChannelHandlerContext ctx, 
        SocketAddress localAddress, 
        ChannelPromise promise
    ) throws Exception;
    void connect(
        ChannelHandlerContext ctx, 
        SocketAddress remoteAddress,
        SocketAddress localAddress, 
        ChannelPromise promise) throws Exception;
    void disconnect(
        ChannelHandlerContext ctx, 
        ChannelPromise promise) throws Exception;
    void close(
        ChannelHandlerContext ctx, 
        ChannelPromise promise) throws Exception;
    
    /**
     * 插销 Channel 注册到 EventLoop 的操作发生时调用该方法
     */
    void deregister(
        ChannelHandlerContext ctx, 
        ChannelPromise promise) throws Exception;

    /**
     * 该方法拦截 ChannelHandlerContext 中的 read()方法
     */
    void read(ChannelHandlerContext ctx) throws Exception;

    /**
     * 向通信对端（客户端或服务器）发送数据时调用该方法
     */
    void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception;

    /**
     * flush 上面的 write 方法写的数据
     */
    void flush(ChannelHandlerContext ctx) throws Exception;
}
```

ChannelInboundHandler 的源码也很简单，如下：

```java
public interface ChannelInboundHandler extends ChannelHandler {

    /**
     * Handler 对应的 Channel 被注册到 EventLoop 上时被调用
     */
    void channelRegistered(ChannelHandlerContext ctx) throws Exception;

    /**
     * Handler 对应的 Channel 从 EventLoop 上解除注册时被调用
     */
    void channelUnregistered(ChannelHandlerContext ctx) throws Exception;

    /**
     * Handler 对应的 Channel 处于活动状态时被调用
     */
    void channelActive(ChannelHandlerContext ctx) throws Exception;

    /**
     * Handler 对应的 Channel 处于非活动状态时被调用
     */
    void channelInactive(ChannelHandlerContext ctx) throws Exception;

    /**
     * 读 Handler 对应的 Channel 中的数据时，该方法被调用
     */
    void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;

    /**
     * 读 Handler 对应的 Channel 中的数据完毕时，该方法被调用
     */
    void channelReadComplete(ChannelHandlerContext ctx) throws Exception;

    ......
}
```

ChannelInboundHandler 或 ChannelOutboundHandler 这两个接口是每个自定义 Handler 要实现的，这两个接口像是一种协议，约定了每个 Handler 要处理哪些工作。当然，为了方便使用者开发自己的 Handler，Netty 提供了一些 Handler 半成品，比如 ChannelInboundHandlerAdapter、ChannelOutboundHandlerAdapter、ChannelDuplexHandler 这三个类，使用者只需继承这些半成品类然后重写某些方法即可（这样就不用逐个实现 ChannelInboundHandler 或 ChannelOutboundHandler 中的全部方法）。请读者自行阅读这些半成品的源码，他们仅仅对 ChannelInboundHandler 或 ChannelOutboundHandler 做了简单的实现。

## 5. ChannelHandlerContext 源码剖析

这一节对 ChannelHandlerContext 的源码进行梳理和总结。

ChannelHandlerContext 是一个接口，并且继承了 AttributeMap、ChannelInboundInvoker、ChannelOutboundInvoker（所谓 Invoker 就是触发器/调用器，用来调用 ChannelInboundHandler 或者 ChannelOutboundHandler 中的事件处理方法）三个接口。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/142.png)

因此 ChannelHandlerContext 有三个作用：

1）维护一些额外属性，这个作用继承于 AttributeMap。

```java
public interface AttributeMap {
    /**
     * 根据 AttributeKey 返回 Attribute
     */
    <T> Attribute<T> attr(AttributeKey<T> key);

    /**
     * 检查 AttributeKey 是否存在
     */
    <T> boolean hasAttr(AttributeKey<T> key);
}
```

2）调用其内部的 ChannelInboundHandler 中的入站事件处理方法来处理 IO 事件，这个作用继承于 ChannelInboundInvoker。ChannelInboundInvoker 在 ChannelInboundHandler 外面包装了一层，达到在触发/调用 ChannelInboundHandler 相应方法前后拦截并做一些特定操作的目的。

```java
public interface ChannelInboundInvoker {
    ......

    /**
     * 触发/调用 ChannelInboundHandler#channelActive(ChannelHandlerContext)方法
     */
    ChannelInboundInvoker fireChannelActive();

    /**
     * 触发/调用 ChannelInboundHandler#channelInactive(ChannelHandlerContext)方法
     */
    ChannelInboundInvoker fireChannelInactive();

    /**
     * 触发/调用 ChannelInboundHandler#exceptionCaught(ChannelHandlerContext)方法
     */
    ChannelInboundInvoker fireExceptionCaught(Throwable cause);

    /**
     * 触发/调用 ChannelInboundHandler#channelRead(ChannelHandlerContext)方法
     */
    ChannelInboundInvoker fireChannelRead(Object msg);

    /**
     * 触发/调用 ChannelInboundHandler#channelReadComplete(ChannelHandlerContext)方法
     */
    ChannelInboundInvoker fireChannelReadComplete();
    ......
}
```

3）调用其内部的 ChannelOutboundHandler 中的出站事件处理方法来处理 IO 事件，这个作用继承于 ChannelOutboundInvoker。ChannelOutboundInvoker 在 ChannelOutboundHandler 外面包装了一层，达到在触发/调用 ChannelOutboundHandler 相应方法前后拦截并做一些特定操作的目的。

```java
public interface ChannelOutboundInvoker {

    /**
     * 触发/调用 ChannelOutboundHandler 中的相应方法
     */
    ChannelFuture bind(SocketAddress localAddress);
    ChannelFuture connect(SocketAddress remoteAddress);
    ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress);
    ChannelFuture disconnect();
    ChannelFuture close();
    ChannelFuture deregister();
    ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise);
    ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise);
    ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
    ChannelFuture disconnect(ChannelPromise promise);
    ChannelFuture close(ChannelPromise promise);
    ChannelFuture deregister(ChannelPromise promise);
    ChannelOutboundInvoker read();
    ChannelFuture write(Object msg);
    ChannelFuture write(Object msg, ChannelPromise promise);
    ChannelOutboundInvoker flush();
    ChannelFuture writeAndFlush(Object msg, ChannelPromise promise);
    ChannelFuture writeAndFlush(Object msg);
    ......
}
```

除了以上从 AttributeMap、ChannelInboundInvoker、ChannelOutboundInvoker 继承来的三个作用，ChannelHandlerContext 还有一些自有方法，如下：

```java
public interface ChannelHandlerContext 
    extends AttributeMap, 
            ChannelInboundInvoker, 
            ChannelOutboundInvoker {

    /**
     * 获取当前 Context 关联的的 Channel
     */
    Channel channel();

    /**
     * 获取当前 Context 关联的的 EventExecutor
     */
    EventExecutor executor();

    ......

    /**
     * 获取当前 Context 关联的的 ChannelHandler
     */
    ChannelHandler handler();

    /**
     * 检测当前 Context 关联的 ChannelHandler 是否从 Pipeline 中被移除了
     */
    boolean isRemoved();
    ......

    /**
     * 获取当前 Context 关联的的 ChannelPipeline
     */
    ChannelPipeline pipeline();

    /**
     * 获取当前 Context 关联的的 ByteBufAllocator
     */
    ByteBufAllocator alloc();

    ......
}
```

总之，ChannelHandlerContext 包装了 ChannelHandler 的一切，以方便 Pipeline 使用 ChannelHandler。

## 6. ChannelPipeline、ChannelHandler、ChannelHandlerContext 三者的创建过程源码剖析

这一节对 ChannelPipeline、ChannelHandler、ChannelHandlerContext 三者的创建过程进行梳理和总结。如下：

1）任何一个 ChannelSocket（无论是 NioSocketChannel 还是 NioServerSocketChannel）创建的时候都会创建一个 ChannelPipeline，这发生在调用 AbstractChannel（它是 NioSocketChannel 和 NioServerSocketChannel 的父类）的构造方法的时候。如下图所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/143.png)

2）当系统内部或者使用者调用 ChannelPipeline 的 addxxx 方法添加 new 出来的 ChannelHandler 实例的时候，都会创建一个包装该 ChannelHandler 的 ChannelHandlerContext。如下图所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/144.png)

3）这些 ChannelHandlerContext 构成了 ChannelPipeline 中的双向循环链表。

## 7. ChannelPipeline 调用 ChannelHandler 源码剖析

这一节对 ChannelPipeline 调用 ChannelHandler 的过程进行梳理和总结。如下：

当一个请求进来的时候，EventLoop 中的 run 方法中的死循环中的 processSelectedKeys，会触发/调用 Pipeline 中的相关方法来处理。如果是处理入站事件，则这些方法由 fire 开头（如 fireChannelRead），表示开始事件在管道中的流动，让后面的 InboundHandler 逐个处理。当处理完业务，要进行响应数据发送以及关闭连接等操作的时候，即处理出站事件的时候，同样要触发/调用 Pipeline 中的相关方法来处理，如 write 方法。

在本文使用的 EchoServer/EchoClient 案例中，Netty 为每一个 NioServerSocketChannel 和 NioSocketChannel 创建的 ChannelPipeline 的真实类型都是 DefaultChannelPipeline，它对 ChannelPipeline 接口进行了实现。比如其中的 addLast 方法、fireChannelRead 方法和各种 write 方法的实现为：

```java
// 以下代码段摘自 DefaultChannelPipeline 源码

......

@Override
public final ChannelPipeline addLast(
    String name, 
    ChannelHandler handler) {
    return addLast(null, name, handler);
}

@Override
public final ChannelPipeline addLast(
    EventExecutorGroup group, 
    String name, 
    ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    // 对操作双线循环链表的代码做同步处理
    synchronized (this) {
        // 检查 handler 是否能被多个 pipeline 重复使用
        checkMultiplicity(handler);
        // 创建包装 handler 的 context
        newCtx = newContext(group, filterName(name, handler), handler);
        // 添加 context 到双向循环链表
        addLast0(newCtx);

        ......
    }
    ......
}

private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
......

@Override
public final ChannelPipeline fireChannelRead(Object msg) {
    // 从双向链表的头部节点开始，逐个寻找下一个 InboundHandler 处理 Read 事件
    // 入站事件的流向为从前往后
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}
......

@Override
public final ChannelFuture write(Object msg) {
    // 从双向链表的尾部节点开始，逐个寻找下一个 OutboundHandler 处理 write 事件
    // 出站事件的流向为从后往前
    return tail.write(msg);
}

@Override
public final ChannelFuture write(Object msg, ChannelPromise promise) {
    // 从双向链表的尾部节点开始，逐个寻找下一个 OutboundHandler 处理 write 事件
    // 出站事件的流向为从后往前
    return tail.write(msg, promise);
}

@Override
public final ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    // 从双向链表的尾部节点开始，逐个寻找下一个 OutboundHandler 处理 write 事件
    // 出站事件的流向为从后往前
    return tail.writeAndFlush(msg, promise);
}

@Override
public final ChannelFuture writeAndFlush(Object msg) {
    // 从双向链表的尾部节点开始，逐个寻找下一个 OutboundHandler 处理 write 事件
    // 出站事件的流向为从后往前
    return tail.writeAndFlush(msg);
}
......
```

对这些处理 IO 事件的方法追踪下去，会发现最终都是去调用各种 Handler 中的方法，这就是我们将自定义 Handler 通过 addLast 添加到 Pipeline 中会发挥作用的底层原理。

Pipeline 就像一个串联的插槽，每个具备特定功能的 Handler 组件可以被热插拔到 Pipeline 中。每一个 IO 事件都会流经 Pipeline，遇到适配的 Handler 就会被处理，当前 Handler 处理完成后会继续抛给后一个 Handler 处理。这个过程如下面的流程图所示（该图给出的是处理入站事件的流程，处理出站事件的流程类似）：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/145.png)

## 8. NioEventLoop 源码剖析

这一节对 NioEventLoop 的源码进行梳理和总结。

我们先来看下 NioEventLoop 的继承关系图。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/146.png)

1）NioEventLoop 实现了 EventLoop 接口，因此它要完成处理注册其上的 Channel 的所有 IO 操作的工作，这是 EventLoop 接口的要求（这里的 EventLoop 是一种职能声明接口，它并没有约定实现类应该具备哪些方法以完成对 IO 事件的处理）。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/148.png)

2）NioEventLoop 间接继承于 ScheduledExecutorService，它是一个定时任务接口，因此 EventLoop 具备接受定时任务的功能，在[上一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)展示向定时任务队列中添加定时任务以实现异步处理时使用的 eventloop 的 schedule 方法也继承于此。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/147.png)

3）NioEventLoop 间接继承于 SingleThreadEventExecutor，也就是说 NioEventLoop 是一个包含单线程的线程池。我们在前文频繁提到的 eventloop 的 runAllTasks 方法就继承于这里。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/150.png)

同样，在[上一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)展示向任务队列中添加任务以实现异步处理时使用的 eventloop 的 execute 方法也继承于此。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/152.png)

4）实际上 NioEventLoop 中的 execute 方法才是 NioEventLoop 中 run 方法（包含了那个核心的死循环过程）的调用者，而 NioEventLoop 中的 execute 方法可以在很多地方被触发，例如 EchoClient 启动的时候，向 EchoServer 发起连接建立请求，EchoServer 创建 NioSocketChannel 并将其注册到 NioEventLoop 上的时候，会触发 NioEventLoop 中的 execute 方法的执行。过程如下：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/149.png)

此外，向任务队列添加任务、添加 Handler 到 Pipeline 等等过程都会触发 NioEventLoop 中 execute 方法的执行。

5）NioEventLoop 中的 execute 方法的执行轨迹如下图所示：

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/151.png)

6）以上 run 方法中，第一步 select 的目的是调用 Java NIO 中 Selector 中的 select()或者 selectNow()方法，把注册到当前 NioEventLoop 中发生 IO 事件的 Channel 对应的 SelectionKey 保存到 Selector 的内部集合中（关于 Java NIO 中 Selector 中的 select()或者 selectNow()方法的作用，可以参考[上一篇文章](https://mp.weixin.qq.com/s/kUkw-RoqLEEr1xuv2ex0FQ)，返回注册到当前 NioEventLoop 中的发生 IO 事件的 Channel 的个数。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/153.png)

第二步 processSelectedKeys 的目的是遍历 Selector 的内部 SelectionKey 集合（每一个 SelectionKey 关联了发生 IO 事件的 Channel），对每一个 Channel 上的 IO 事件进行处理。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/154.png)

第三步 runAllTasks 的目的是处理任务队列中的异步任务。任务队列中的任务可以是第二步处理 IO 事件的时候添加的，以实现处理过程的异步化。

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/155.png)

## 结束语

本文第 1、2 两节通过追踪 Netty 服务端启动过程和接收请求过程的源码，展示 Netty 中 ServerBootStrap、EventLoopGroup、EventLoop、ChannelPipeline、ChannelHandler、ChannelHandlerContext 这些核心组件的工作原理，后面几节又对部分组件的源码做了专项的梳理和总结，以帮助读者加深对 Netty 架构和核心组件的理解。那再次回顾这张 Netty 的架构图，你是否看到了更多的细节呢？

![](https://gitee.com/guo_keyan2/pic_pic/raw/master/img/104.png)

下一篇文章，《Netty 三讲》的第三讲：**Netty 的应用案例**，我将给大家分享一些基于 Netty 开发网络 IO 应用的关键方法和案例。

------

**参考资料：**

1. Netty 官网文档，https://netty.io/wiki/all-documents.html
2. 《Netty 权威指南（第一版）》，李林锋
3. 《Netty in Action》，Norman Maurer
4. 《Scalable IO in Java》，Doug Lea
5. 尚硅谷 Netty 系列教程，韩顺平主讲

最后欢迎大家关注公号「码海」,一起交流，共同进步！

![](https://user-gold-cdn.xitu.io/2019/12/29/16f51ecd24e85b62?w=1002&h=270&f=jpeg&s=59118)
