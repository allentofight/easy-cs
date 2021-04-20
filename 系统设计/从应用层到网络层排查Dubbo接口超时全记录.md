我们常说面试造火箭，很多人对此提出质疑，相信大家看了这篇文章会明白面试造火箭的道理，这篇排查问题的技巧涉及到索引，GC，容器，网络抓包，全链路追踪等基本技能，没有这些造火箭的本事，排查这类问题往往会无从下手，本篇也能回答不少朋友的问题：为什么学 Java 却要掌握网络，MySQL等其他知识体系，这会让你成为更出色的工程师哦。


## 一. 问题现象

商品团队反馈，会员部分 dubbo 接口偶现超时异常，而且时间不规律，几乎每天都有，商品服务超时报错如下图：


![](https://tva1.sinaimg.cn/large/008eGmZEly1gppyik4xc4j31kw0i2agj.jpg)

超时的接口平时耗时极短，平均耗时 4-5 毫秒。查看 dubbo 的接口配置，商品调用会员接口超时时间是一秒，失败策略是failfast，快速失败不会重试。

会员共部署了8台机器，都是 Java 应用，Java 版本使用的是 JDK 8，都跑在 docker 容器中 。


## 二. 问题分析

开始以为只是简单的接口超时，着手排查。首先查看接口逻辑，只有简单的数据库调用，封装参数返回。SQL 走了索引查询，理应返回很快才是。

于是搜索 dubbo 的拦截器 ElapsedFilter 打印的耗时日志（ElapsedFilter 是 dubbo SPI 机制的扩展点之一，超过 300 毫秒的接口耗时都会打印），这个接口的部分时间耗时确实很长。

![](https://tva1.sinaimg.cn/large/008eGmZEly1gpq4gvpqvrj30qb0esgp5.jpg)

再查询数据库的慢 SQL 平台，未发现慢 SQL。



### 1. 数据库超时？

怀疑是获取数据库连接耗时。代码中调用数据库查询可以分为两步，获取数据库连接和 SQL 查询，慢SQL 平台监测的 SQL 执行时间，既然慢 SQL 平台没有发现，那可能是建立连接耗时较长。

查看应用中使用了 Druid 连接池，数据库连接会在初始化使用时建立，而不是每次 SQL 查询再建立连接。Druid 配置的 initial-size（初始化链接数）是 5，max-active（最大连接池数量）是 50，但 min-idle（最小连接池数量）只有 1，猜测可能是连接长时间不被使用后都回收了，有新的请求进来后触发建立连接，耗时较长。

因此调整 min-idle 为 10，并加上了 Druid 连接池的使用情况日志，每隔 3 秒打印连接池的活跃数等信息，重新观察。

改动上线后，还是有超时发生，超时发生时查看连接池日志，连接数正常，连接池连接数也没什么波动。

### 2. STW？

因此重新回到报错日志观察，ElapsedFilter 打印的耗时日志已经消失。由于在业务方法的入口和出口，以及数据库操作的前后打印了日志，发现整个业务操作耗时极短，都在几毫秒内完成。

但是调用端隔一段时间开始超时报错，且并不是调用一个接口超时，而是调用好几个接口都同时超时。另外，调用端的几台机器都会同时报超时（可以在 elk 平台筛选 hostname 查看），而提供服务超时的机器是同一台。

这样问题就比较清晰了，**应该是某一时刻提供服务的某台机器出现比较全局性的问题导致几乎所有接口都超时**。

继续观察日志，抓了其中一个超时的请求从调用端到服务端的所有日志（理应有分布式 ID 可以追踪，context id 只能追踪单应用内的一个请求，跨应用就失效了，所以只能自己想办法，此处是根据调用IP+时间+接口+参数在 elk 定位）。
这次有了新的发现，服务端收到请求的时间比调用端发起调用的时间晚了一秒，比较严谨的说法是调用端的超时日志中 dubbo 有打印 startTime（记为T1）和 endTime，同时在服务端的接口方法入口加了日志，可以定位请求进来的时间（记为T2），比较这两个时间，发现 T2 比 T1 晚了一秒多，更长一点的有两到三秒。
内网调用网络时间几乎可以忽略不计，这个延迟时间极其不正常。很容易想到 Java 应用的 STW(Stop The World)，其中，特别是垃圾回收会导致短暂的应用停顿，无法响应请求。

排查垃圾回收问题第一件事自然是打开垃圾回收日志（GC log），GC log 打印了 GC 发生的时间，GC 的类型，以及 GC 耗费的时间等。增加 JVM 启动参数

>  -XX:+PrintGCDetails -Xloggc:${APPLICATION_LOG_DIR}/gc.log -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime。

为了方便排查，同时打印了所有应用的停顿时间。（除了垃圾回收会导致应用停顿，还有很多操作也会导致停顿，比如取消偏向锁等操作）。由于应用是在 docker 环境，因此每次应用发布都会导致 GC 日志被清除，写了个上报程序，定时把 GC log 上报到 elk 平台，方便观察。

以下是接口超时时 gc 的情况：

![](https://tva1.sinaimg.cn/large/008eGmZEly1gppykmybq4j31z20cs1dh.jpg)

可以看到，有次 GC 耗费的时间接近两秒，应用的停顿时间也接近两秒，而且此次的垃圾回收是ParNew 算法，也就是发生在新生代。所以基本可以确定，**是垃圾回收的停顿导致应用不可用，进而导致接口超时**。

### 2.1 安全点及 FinalReferecne

以下开始排查新生代垃圾回收耗时过长的问题。
首先，应用的 JVM 参数配置是

>  -Xmx5g -Xms5g -Xmn1g -XX:MetaspaceSize=512m -XX:MaxMetaspaceSize=512m -Xss256k -XX:SurvivorRatio=8 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=70

通过观察 marvin 平台（我们的监控平台），发现堆的利用率一直很低，GC 日志甚至没出现过一次 CMS GC，由此可以排除是堆大小不够用的问题。

ParNew 回收算法较为简单，不像老年代使用的 CMS 那么复杂。ParNew 采用标记-复制算法，把新生代分为 Eden 和 Survivor 区，Survivor 区分为 S0 和 S1，新的对象先被分配到 Eden 和 S0 区，都满了无法分配的时候把存活的对象复制到 S1 区，清除剩下的对象，腾出空间。比较耗时的操作有三个地方，**找到并标记存活对象，回收对象，复制存活对象**（此处涉及到比较多垃圾回收的知识，详细部分就不展开说了）。

找到并标记存活对象前 JVM 需要暂停所有线程，这一步并不是一蹴而就的，需要等所有的线程都跑到安全点。部分线程可能由于执行较长的循环等操作无法马上响应安全点请求，JVM 有个参数可以打印所有安全点操作的耗时，加上参数

>  -XX:+PrintSafepointStatistics -XX: PrintSafepointStatisticsCount=1 -XX:+UnlockDiagnosticVMOptions -XX: -DisplayVMOutput -XX:+LogVMOutput -XX:LogFile=/opt/apps/logs/safepoint.log。

![](https://tva1.sinaimg.cn/large/008eGmZEly1gppynm6kcej31ks0foh00.jpg)

安全点日志中 spin+block 时间是线程跑到安全点并响应的时间，从日志可以看出，此处耗时极短，大部分时间都消耗在 vmop 操作，也就是到达安全点之后的操作时间，排除线程等待进入安全点耗时过长的可能。

继续分析回收对象的耗时，根据引用强弱程度不通，Java 的对象类型可分为各类 Reference，其中FinalReference 较为特殊，对象有个 finalize 方法，可在对象被回收之前调用，给对象一个垂死挣扎的机会。有些框架会在对象 finalize 方法中做一些资源回收，关闭连接的操作，导致垃圾回收耗时增加。因此通过增加JVM参数 -XX:+PrintReferenceGC，打印各类 Reference 回收的时间。

![](https://tva1.sinaimg.cn/large/008eGmZEly1gppyoh0id4j31h405bgs9.jpg)

可以看到，FinalReference 的回收时间也极短。

最后看复制存活对象的耗时，复制的时间主要由存活下来的对象大小决定，从 GC log 可以看到，每次新生代回收基本可以回收百分之九十以上的对象，存活对象极少。因此基本可以排除这种可能。
问题的排查陷入僵局，ParNew 算法较为简单，因此 JVM 并没有更多的日志记录，所以排查更多是通过经验。

### 2.2 垃圾回收线程

不得已，开始求助网友，上 stackoverflow 发帖（链接见文末），有大神从 GC log 发现，有两次 GC 回收的对象大小几乎一样，但是耗时却相差十倍，因此建议确认下系统的 CPU 情况，是否是虚拟环境，可能存在激烈的 CPU 资源竞争。

其实之前已经怀疑是 CPU 资源问题，通过 marvin 监控发现，垃圾回收时 CPU 波动并不大。但运维发现我们应用的垃圾回收线程数有些问题（GC 线程数可以通过 jstack 命令打印线程堆栈查看），JVM 的 GC 线程数量是根据 CPU 的核数确定的，如果是八个核心以下，GC 线程数等于CPU 核心数，我们应用跑在 docker 容器，分配的核心是六核，但是新生代 GC 线程数却达到了 53个，这明显不正常。
最后发现，这个问题是 JVM 在 docker 环境中，获取到的 CPU 信息是宿主机的（容器中 /proc目录并未做隔离，各容器共享，CPU信息保存在该目录下），并不是指定的六核心，宿主机是八十核心，因此创建的垃圾回收线程数远大于 docker 环境的 CPU 资源，导致每次新生代回收线程间竞争激烈，耗时增加。

通过指定 JVM 垃圾回收线程数为 CPU 核心数，限制新生代垃圾回收线程，增加JVM参数

>  -XX:ParallelGCThreads=6 -XX:ConcGCThreads=6


![](https://tva1.sinaimg.cn/large/008eGmZEly1gppyrg6oolj30zq0b0mye.jpg)

效果立竿见影！新生代垃圾回收时间基本降到 50 毫秒以下，成功解决垃圾回收耗时较长问题。

## 三. 问题重现


本以为超时问题应该彻底解决了，但还是收到了接口超时的报警。现象完全一样，同一时间多个接口同时超时。查看 GC 日志，发现已经没有超过一秒的停顿时间，甚至上百毫秒的都已经没有。



### 1. Arthas 和 Dubbo

回到代码，重新分析。开始对整个 Dubbo 接口接收请求过程进行拆解，打算借助阿里巴巴开源的Arthas 对请求进行 trace，打印全链路各个步骤的时间。（**Arthas 和 dubbo**）
服务端接收 dubbo 请求，从网络层开始再到应用层。具体是从 netty 的 worker 线程接收到请求，再投递到 dubbo 的业务线程池（应用使用的是 ALL 类型线程池），由于涉及到多个线程，只能分两段进行 trace，netty 接收请求的一段，dubbo 线程处理的一段。
Arthas 的 trace 功能可以在后台运行，同时只打印耗时超过某个阈值的链路。（trace 采用的是instrument+ASM，会导致短暂的应用暂停，暂停时间取决于被植入的类的数量，需注意）
由于没有打印参数，无法准确定位超时的请求，而且 trace 只能看到调用第一层的耗时时间，结果都是业务处理时间过长，最后放弃了该trace方法。

受 GC 线程数思路的启发，由于应用运行基本不涉及刷盘的操作，都是 CPU 运算+内存读取，耗时仍旧应该是线程竞争或者锁引起的。重新回到 jstack 的堆栈进行排查，发现总线程数仍有 900+，处于一个较高的水平，其中 forkjoin 线程和 netty 的 worker 的线程数仍较多。于是重新搜索代码中线程数是通过CPU核心数设置（Runtime.getRuntime().availableProcessors()）的地方，发现还是有不少框架使用这个参数，这部分无法通过设置JVM参数解决。

因此和容器组商量，是否能够给容器内应用传递正确的 CPU 核心数。容器组确认可以通过升级 JDK 版本和开启 CPU SET 技术限制容器使用的 CPU 数，从 Java 8u131 和 Java 9 开始，JVM 可以理解和利用 cpusets 来确定可用处理器的大小(**Java 和 cpu set 见文末参考链接**)。现在使用的也是JDK 8，小版本升级风险较小，因此测试没问题后，推动应用内8台容器升级到了 Java 8u221，并且开启了 cpu set，指定容器可使用的 CPU 数。
重新修改上线后，还是有超时情况发生。

### 2. 网络抓包

观察具体的超时机器和时间，发现并不是多台机器超时，而是一段时间有某台机器比较密集的超时，而且频率比之前密集了许多。以前的超时在各机器之间较为随机，而且频率低很多。比较奇怪的是，wukong 经过发布之后，原来超时的机器不超时了，而是其他的机器开始超时，看起来像是接力赛。（docker 应用的宿主机随着每次发布都可能变化）

面对如此奇怪的现象，只能怀疑是宿主机或者是网络问题了，因此开始网络抓包。

由于超时是发生在一台机器上，而且频率较为密集，因此直接进入该机器，使用 tcpdump 命令抓取host 是调用方进来的包并保存抓包记录，通过 wireshark 分析（wireshark 是抓包分析神器），抓到了一些异常包。
![](https://tva1.sinaimg.cn/large/008eGmZEly1gppyuq5napj321c0fek5k.jpg)
> 注意抓包中出现的 「TCP Out_of_Order」,「TCP Dup ACK」,「TCP Retransmission」，三者的释义如下:
> 
>`TCP Out_of_Order`:  一般来说是网络拥塞，导致顺序包抵达时间不同，延时太长，或者包丢失，需要重新组合数据单元，因为他们可能是由不同的路径到达你的电脑上面。 
> 
>`TCP dup ack XXX#X`: 重复应答，#前的表示报文到哪个序号丢失，#后面的是表示第几次丢失
> 
>`TCP Retransmission`： 超时引发的数据重传
终于，通过在 elk 中超时日志打印的 contextid，在 wireshark 过滤定位超时的 TCP 包，发现超时的时候有丢包和重传。抓了多个超时的日志都有丢包发生，确认是网络问题引起。

问题至此基本告一段落，接下来容器组继续排查网络偶发丢包问题。


巨人的肩膀

* stackoverflow 提问帖子: http://aakn.cn/YbFsl
* jstack 可视化分析工具：https://fastthread.io/
* 2018年的 Docker 和 JVM：`https://www.jdon.com/51214

欢迎关注公众号与笔者共同交流哦^_^

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a18df9c2b2d24604a18f3d85cd409ca3~tplv-k3u1fbpfcp-zoom-1.image)
