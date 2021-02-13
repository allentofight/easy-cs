# 一、背景介绍
近一年内对公司的 ELK 日志系统做过性能优化，也对 SkyWalking 使用的 ES 存储进行过性能优化，在此做一些总结。本篇主要是讲 ES 在 ELK 架构中作为日志存储时的性能优化方案。

####  ELK 架构作为日志存储方案
![ELK日志架构.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2958ab55f78c441f94da3ee9e954c8c3~tplv-k3u1fbpfcp-zoom-1.image)

# 二、现状分析
## 1. 版本及硬件配置
* **JDK**：JDK1.8_171-b11 (64位)
* **ES集群**：由3台16核32G的虚拟机部署 ES 集群，每个节点分配20G堆内存
* **ELK版本**：6.3.0
* **垃圾回收器**：ES 默认指定的老年代（CMS）+ 新生代（ParNew）
* **操作系统**：CentOS Linux release 7.4.1708(Core)

## 2. 性能问题
随着接入 ELK 的应用越来越多，**每日新增索引约 230 个，新增 document 约 3000 万到 5000 万**。

每日上午和下午是日志上传高峰期，在 Kibana 上查看日志，发现问题：

**(1) 日志会有 5-40 分钟的延迟**

**(2) 有很多日志丢失，无法查到**


## 3. 问题分析
### 3.1 日志延迟
##### 首先了解清楚：数据什么时候可以被查到？
数据先是存放在 ES 的内存 buffer，然后执行 refresh 操作写入到操作系统的内存缓存 os cache，此后数据就可以被搜索到。

所以，日志延迟可能是我们的数据积压在 buffer 中没有进入 os cache 。

### 3.2日志丢失

查看日志发现很多 write 拒绝执行的情况

![write 线程池拒绝任务的日志.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e77be153caf478eb40c351e17aefe97~tplv-k3u1fbpfcp-zoom-1.image)

从日志中可以看出 ES 的 write 线程池已经满负荷，执行任务的线程已经达到最大16个线程，而200容量的队列也已经放不下新的task。



查看线程池的情况也可以看出 write 线程池有很多写入的任务

```
GET /_cat/thread_pool?v&h=host,name,type,size,active,largest,rejected,completed,queue,queue_size
```

![write 线程池拒绝任务的情况.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7fb13919842460c8439e72095a905d0~tplv-k3u1fbpfcp-zoom-1.image)



所以我们需要优化 ES 的 write 的性能。




### 4.解决思路
#### 4.1 分析场景
ES 的优化分为很多方面，我们要根据使用场景考虑对 ES 的要求。

**根据个人实践经验，列举三种不同场景下的特点**：

* **SkyWalking**：一般配套使用 ES 作为数据存储，存储链路追踪数据、指标数据等信息。
* **ELK**：一般用来存储系统日志，并进行分析，搜索，定位应用的问题。
* **全文搜索的业务**：业务中常用 ES 作为全文搜索引擎，例如在外卖应用中，ES 用来存储商家、美食的业务数据，用户在客户端可以根据关键字、地理位置等查询条件搜索商家、美食信息。

**这三类场景的特点如下：**

|            | SkyWalking         | ELK                | 全文搜索的业务     |
| ---------- | ------------------ | ------------------ | ------------------ |
| 并发写     | 高并发写           | 高并发写           | 并发一般不高       |
| 并发读     | 并发低             | 并发低             | 并发高             |
| 实时性要求 | 5分钟以内          | 30秒以内           | 1分钟内            |
| 数据完整性 | 可容忍丢失少量数据 | 可容忍丢失少量数据 | 数据尽量100%不丢失 |

**关于实时性**
* SkyWalking 在实际使用中，一般使用频率不太高，往往是发现应用的问题后，再去 SkyWalking 查历史链路追踪数据或指标数据，所以可以接受几分钟的延迟。
* ELK 不管是开发、测试等阶段，时常用来定位应用的问题，如果不能快速查询出数据，延迟太久，会耽误很多时间，大大降低工作效率；如果是查日志定位生产问题，那更是刻不容缓。
* 全文搜索的业务中一般可以接受在1分钟内查看到最新数据，比如新商品上架一分钟后才看到，但尽量实时，在几秒内可以可看到。


#### 4.2 优化的方向
可以从三方面进行优化：JVM性能调优、ES性能调优、控制数据来源

# 三、ES性能优化
可以从三方面进行优化：JVM 性能调优、ES 性能调优、控制数据来源
## 1. JVM调优

**第一步是 JVM 调优。**

因为 ES 是依赖于 JVM 运行，没有合理的设置 JVM 参数，将浪费资源，甚至导致 ES 很容易 OOM 而崩溃。

### 1.1 监控 JVM 运行情况
**(1) 查看 GC 日志**
![GC日志中很多Young GC.png](https://i.loli.net/2021/01/07/UxSNGF6VMtOermL.png)
**问题**：Young GC 和 Full GC 都很频繁，特别是 Young GC 频率高，累积耗时非常多。

**(2) 使用 jstat 看下每秒的 GC 情况**
![jstat发现Young GC频繁.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50b407053e8b4dcda5b88235f373608b~tplv-k3u1fbpfcp-zoom-1.image)
**参数说明**

* S0：幸存1区当前使用比例
* S1：幸存2区当前使用比例
* E：伊甸园区使用比例
* O：老年代使用比例
* M：元数据区使用比例
* CCS：压缩使用比例
* YGC：年轻代垃圾回收次数
* FGC：老年代垃圾回收次数
* FGCT：老年代垃圾回收消耗时间
* GCT：垃圾回收消耗总时间
**问题**：从 jstat gc 中也可以看出，每秒的 eden 增长速度非常快，很快就满了。


### 1.2 定位 Young GC 频繁的原因
#### 1.2.1 检查是否新生代的空间是否太小
用下面几种方式都可查看新、老年代内存大小
(1) 使用 **jstat -gc pid**  查看 Eden 区、老年代空间大小
(2) 使用 **jmap -heap pid**  查看 Eden 区、老年代空间大小
(3) 查看 GC 日志中的 GC 明细
![GC明细.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/71188c97d9be4239bfbdac5595c6b48e~tplv-k3u1fbpfcp-zoom-1.image)
其中 996800K 为新生代可用空间大小，即 Eden 区 +1 个 Survivor 区的空间大小，所以新生代总内存是996800K/0.9， 约1081M

上面的几种方式都查询出，新生代总内存约1081M，即1G左右；老年代总内存为19864000K，约19G。新、老比例约1:19，出乎意料。

#### 1.2.1 新老年代空间比例为什么不是 JDK 默认的1:2【重点！】
**这真是一个容易踩坑的地方。**
如果没有显示设置新生代大小，JVM 在使用 CMS 收集器时会自动调参，新生代的大小在没有设置的情况下是通过计算得出的，其大小可能与 NewRatio 的默认配置没什么关系而与 ParallelGCThreads 的配置有一定的关系。

> 参考文末链接：[CMS GC 默认新生代是多大？](https://www.jianshu.com/p/832fc4d4cb53)

所以：**新生代大小有不确定性，最好配置 JVM 参数 -XX:NewSize、-XX:MaxNewSize 或者 -xmn ，免得遇到一些奇怪的 GC，让人措手不及。**

### 1.3 上面现象造成的影响
**新生代过小，老年代过大的影响**

* **新生代过小**:
  (1) 会导致新生代 Eden 区很快用完，而触发 Young GC，Young GC 的过程中会 STW(Stop The World)，也就是所有工作线程停止，只有 GC 的线程在进行垃圾回收，这会导致 ES 短时间停顿。频繁的 Young GC，积少成多，对系统性能影响较大。
(2) 大部分对象很快进入老年代，老年代很容易用完而触发 Full GC。
* **老年代过大**：会导致 Full GC 的执行时间过长，Full GC 虽然有并行处理的步骤，但是还是比 Young GC 的 STW 时间更久，而 GC 导致的停顿时间在几十毫秒到几秒内，很影响 ES 的性能，同时也会导致请求 ES 服务端的客户端在一定时间内没有响应而发生 timeout 异常，导致请求失败。


### 1.4 JVM优化
#### 1.4.1 配置堆内存空间大小
32G 的内存，分配 20G 给堆内存是不妥当的，所以调整为总内存的50%，即16G。
修改 elasticsearch 的 jvm.options 文件

```
-Xms16g
-Xmx16g
```

**设置要求:**

* Xms 与 Xmx 大小相同。

  在 jvm 的参数中 -Xms 和 -Xmx 设置的不一致，在初始化时只会初始 -Xms 大小的空间存储信息，每当空间不够用时再向操作系统申请，这样的话必然要进行一次 GC，GC会带来 STW。而剩余空间很多时，会触发缩容。再次不够用时再扩容，如此反复，这些过程会影响系统性能。同理在 MetaSpace 区也有类似的问题。

* jvm 建议不要超过 32G，否则 jvm 会禁用内存对象指针压缩技术，造成内存浪费

* Xmx 和 Xms 不要超过物理 RAM 的50%。 参考文末：[官方堆内存设置的建议](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#heap-size-settings)

  > Xmx 和 Xms 不要超过物理内存的50%。Elasticsearch 需要内存用于JVM堆以外的其他用途，为此留出空间非常重要。例如，Elasticsearch 使用堆外缓冲区进行有效的网络通信，依靠操作系统的文件系统缓存来高效地访问文件，而 JVM 本身也需要一些内存。

#### 1.4.2 配置堆内存新生代空间大小

因为指定新生代空间大小，导致 JVM 自动调参只分配了 1G 内存给新生代。

修改 elasticsearch 的 jvm.options 文件，加上
```
-XX:NewSize=8G
-XX:MaxNewSize=8G
```
老年代则自动分配 16G-8G=8G 内存，新生代老年代的比例为 1:1。修改后每次 Young GC 频率更低，且每次 GC 后只有少数数据会进入老年代。



#### 2.3 使用G1垃圾回收器(未实践)

> G1垃圾回收器让系统使用者来设定垃圾回收堆系统的影响，然后把内存拆分为大量的小 Region，追踪每个 Region 中可以回收的对象大小和回收完成的预计花费的时间， 最后在垃圾回收的时候，尽量把垃圾回收对系统造成的影响控制在我们指定的时间范围内，同时在有限的时间内尽量回收更多的垃圾对象。
> G1垃圾回收器一般在大数量、大内存的情况下有更好的性能。

ES默认使用的垃圾回收器是：老年代（CMS）+ 新生代（ParNew）。如果是JDK1.9，ES 默认使用 G1 垃圾回收器。

因为使用的是 JDK1.8，所以并未切换垃圾回收器。后续如果再有性能问题再切换G1垃圾回收器，测试是否有更好的性能。



### 1.5 优化的效果

#### 1.5.1 新生代使用内存的增长率更低

**优化前**

![优化前.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/173637dcabff4a9eb0eef39282ddbf1a~tplv-k3u1fbpfcp-zoom-1.image)

每秒打印一次 GC 数据。可以看出，年轻代增长速度很快，几秒钟年轻代就满了，导致 Young GC 触发很频繁，几秒钟就会触发一次。而每次 Young GC 很大可能有存活对象进入老年代，而且，存活对象多的时候(看上图中第一个红框中的old gc数据)，有(51.44-51.08)/100 * 19000M = 约68M。每次进入老年代的对象较多，加上频繁的 Young GC，会导致新老年代的分代模式失去了作用，相当于老年代取代了新生代来存放近期内生成的对象。当老年代满了，触发 Full GC，存活的对象也会很多，因为这些对象很可能还是近期加入的，还存活着，所以一次 Full GC 回收对象不多。而这会恶性循环，老年代很快又满了，又 Full GC，又残留一大部分存活的，又很容易满了，所以导致一直频繁 Full GC。



**优化后**

![优化后.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec18eec7b1c24e639fb4fd65168ca0be~tplv-k3u1fbpfcp-zoom-1.image)

每秒打印一次 GC 数据。可以看出，新生代增长速度慢了许多，至少要 60 秒才会满，如上图红框中数据，进入老年代的对象约(15.68-15.60)/100 * 10000 = 8M，非常的少。所以要很久才会触发一次 Full GC 。而且等到 Full GC 时，老年代里很多对象都是存活了很久的，一般都是不会被引用，所以很大一部分会被回收掉，留一个比较干净的老年代空间，可以继续放很多对象。



#### 1.5.2 新生代和老年代 GC 频率更低

ES 启动后，运行14个小时

**优化前**

![优化前.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d2f42b2b76b468cba585944496ea852~tplv-k3u1fbpfcp-zoom-1.image)

Young GC 每次的时间是不长的，从上面监控数据中可以看出每次GC时长 1467.995/27276 约等于 0.05 秒。那一秒钟有多少时间实在处理 Young GC ? 

计算公式：1467 秒/*( 60 秒× 60 分* 14 小时）= 约 0.028 秒，也就是 100 秒中就有 2.8 秒在Young GC，也就是有 2.8S 的停顿，这对性能还是有很大消耗的。同时也可以算出多久一次 Young GC， 方程是： 60秒×60分*14小时/ 27276次 = 1次/X秒，计算得出X = 0.54，也就是 0.54 秒就会有一次Young GC，可见 Young GC 频率非常频繁。

**优化后**

![优化后效果.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae30711d4ead41e89a5946f57b69f34a~tplv-k3u1fbpfcp-zoom-1.image)

Young GC 次数只有修改前的十分之一，Young GC 时间也是约八分之一。Full GC 的次数也是只有原来的八分之一，GC 时间大约是四分之一。

GC 对系统的影响大大降低，性能已经得到很大的提升。



## 2.ES 调优
上面已经分析过 ES 作为日志存储时的特性是：高并发写、读少、接受 30 秒内的延时、可容忍部分日志数据丢失。
下面我们针对这些特性对ES进行调优。

### 2.1 优化 ES 索引设置

#####  2.2.1 ES 写数据底层原理

![ES写入数据的原理.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb011998ecdf462a8270072c06aedb2e~tplv-k3u1fbpfcp-zoom-1.image)

**refresh**
ES 接收数据请求时先存入 ES 的内存中，默认每隔一秒会从内存 buffer 中将数据写入操作系统缓存 os cache，这个过程叫做 refresh；

到了 os cache 数据就能被搜索到（所以我们才说 ES 是近实时的，因为 1 s 的延迟后执行 refresh 便可让数据被搜索到）

**fsync**
translog 会每隔 5 秒或者在一个变更请求完成之后执行一次 fsync 操作，将 translog 从缓存刷入磁盘，这个操作比较耗时，如果对数据一致性要求不是跟高时建议将索引改为异步，如果节点宕机时会有5秒数据丢失;

**flush**
ES 默认每隔30分钟会将 os cache 中的数据刷入磁盘同时清空 translog 日志文件，这个过程叫做 flush。

**merge**

ES 的一个 index 由多个 shard 组成，而一个 shard 其实就是一个 Lucene 的 index ，它又由多个 segment 组成，且 Lucene 会不断地把一些小的 segment 合并成一个大的 segment ，这个过程被称为[段merge](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/index-modules-merge.html)（参考文末链接）。执行索引操作时，[ES会先生成小的segment](https://stackoverflow.com/questions/15426441/understanding-segments-in-elasticsearch)，ES 有离线的逻辑对小的 segment 进行合并，优化查询性能。但是合并过程中会消耗较多磁盘 IO，会影响查询性能。

##### 2.2.2 优化方向

###### 2.2.2.1 优化 fsync

为了保证不丢失数据，就要保护 translog 文件的安全:

> Elasticsearch 2.0 之后, 每次写请求(如 index 、delete、update、bulk 等)完成时, 都会触发`fsync`将 translog 中的 segment 刷到磁盘, 然后才会返回`200 OK`的响应;
>
> 或者: 默认每隔5s就将 translog 中的数据通过`fsync`强制刷新到磁盘.

该方式提高数据安全性的同时， 降低了一点性能.

==> 频繁地执行 `fsync` 操作, 可能会产生阻塞导致部分操作耗时较久. **如果允许部分数据丢失, 可设置异步刷新 translog 来提高效率，还有降低 flush 的阀值，优化如下：**

```
"index.translog.durability": "async",
"index.translog.flush_threshold_size":"1024mb",
"index.translog.sync_interval": "120s"
```

###### 2.2.2.2 优化 refresh

写入 Lucene 的数据，并不是实时可搜索的，ES 必须通过 refresh 的过程把内存中的数据转换成 Lucene 的完整 segment 后，才可以被搜索。

默认 1秒后，写入的数据可以很快被查询到，但势必会产生大量的 segment，检索性能会受到影响。所以，加大时长可以降低系统开销。对于日志搜索来说，实时性要求不是那么高，设置为 5 秒或者 10s；对于 SkyWalking，实时性要求更低一些，我们可以设置为30s。

设置如下：

```
"index.refresh_interval":"5s"
```



###### 2.2.2.3 优化 merge

index.merge.scheduler.max_thread_count 控制并发的 merge 线程数，如果存储是并发性能较好的 SSD，可以用系统默认的 max(1, min(4, availableProcessors / 2))，当节点配置的 cpu 核数较高时，merge 占用的资源可能会偏高，影响集群的性能，普通磁盘的话设为1，发生磁盘 IO 堵塞。设置max_thread_count 后，会有 max_thread_count + 2 个线程同时进行磁盘操作，也就是设置为 1 允许3个线程。

设置如下：

```
"index.merge.scheduler.max_thread_count":"1"
```



##### 2.2.2 优化设置

###### 2.2.2.1 对现有索引做索引设置

```
# 需要先 close 索引,然后再执行,最后成功之后再打开
# 关闭索引
curl -XPOST 'http://localhost:9200/_all/_close'

# 修改索引设置
curl -XPUT -H "Content-Type:application/json" 'http://localhost:9200/_all/_settings?preserve_existing=true' -d '{"index.merge.scheduler.max_thread_count" : "1","index.refresh_interval" : "10s","index.translog.durability" : "async","index.translog.flush_threshold_size":"1024mb","index.translog.sync_interval" : "120s"}'

# 打开索引
curl -XPOST 'http://localhost:9200/_all/_open'
```

该方式可对已经生成的索引做修改，但是对于后续新建的索引不生效，所以我们可以制作 ES 模板，新建的索引按模板创建索引。

###### 2.2.2.2 制作索引模板

```
# 制作模板 大部分索引都是业务应用的日志相关的索引，且索引名称是 202* 这种带着日期的格式
PUT _template/business_log
{
  "index_patterns": ["*202*.*.*"],
  "settings": {
  "index.merge.scheduler.max_thread_count" : "1","index.refresh_interval" : "5s","index.translog.durability" : "async","index.translog.flush_threshold_size":"1024mb","index.translog.sync_interval" : "120s"}
}

# 查询模板是否创建成功
GET _template/business_log
```

因为我们的业务日志是按天维度创建索引，索引名称示例：user-service-prod-2020.12.12，所以用通配符***202*.*.**匹配对应要创建的业务日志索引。



### 2.2 优化线程池配置

前文已经提到过，write 线程池满负荷，导致拒绝任务，而有的数据无法写入。

而经过上面的优化后，拒绝的情况少了很多，但是还是有拒绝任务的情况。

所以我们还需要优化 write 线程池。



**从 prometheus 监控中可以看到线程池的情况：**

为了更直观看到 ES 线程池的运行情况，我们安装了 elasticsearch_exporter 收集 ES 的指标数据到 prometheus，再通过 grafana 进行查看。

经过上面的各种优化，拒绝的数据量少了很多，但是还是存在拒绝的情况，如下图：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb80223934824bcb9bdf867ea0c13711~tplv-k3u1fbpfcp-zoom-1.image)



**write 线程池如何设置：**

参考文末链接：[ElasticSearch线程池](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/modules-threadpool.html#modules-threadpool)

> **`write`**
>
> For single-document index/delete/update and bulk requests. Thread pool type is `fixed` with a size of `# of available processors`, queue_size of `200`. The maximum size for this pool is `1 + # of available processors`.

write 线程池采用 fixed 类型的线程池，也就是核心线程数与最大线程数值相同。线程数默认等于 cpu 核数，可设置的最大值只能是 cpu 核数加 1，也就是 16 核 CPU，能设置的线程数最大值为 17。



**优化的方案：**

* 线程数改为 17，也就是 cpu 总核数加 1
* 队列容量加大。队列在此时的作用是消峰。不过队列容量加大本身不会提升处理速度，只是起到缓冲作用。此外，队列容量也不能太大，否则积压很多任务时会占用过多堆内存。

config/elasticsearch.yml文件增加配置

```
# 线程数设置
thread_pool:
  write:
    # 线程数默认等于cpu核数，即16  
    size: 17
    # 因为任务多时存在任务拒绝的情况，所以加大队列大小，可以在间歇性任务量陡增的情况下，缓存任务在队列，等高峰过去逐步消费完。
    queue_size: 10000
```



**优化后效果**
![线程池拒绝的情况.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1158ff5a424d4b2e95aa2ad293920da1~tplv-k3u1fbpfcp-zoom-1.image)
可以看到，已经没有拒绝的情况，这样也就是解决了日志丢失的问题。



### 2.３ 锁定内存，不让 JVM 使用 Swap

**Swap 交换分区**:

> 当系统的物理内存不够用的时候，就需要将物理内存中的一部分空间释放出来，以供当前运行的程序使用。那些被释放的空间可能来自一些很长时间没有什么操作的程序，**这些被释放的空间被临时保存到 Swap 中，等到那些程序要运行时，再从 Swap 中恢复保存的数据到内存中。**这样，系统总是在物理内存不够时，才进行 Swap 交换。



参考文末链接：[ElasticSearch官方解释为什么要禁用交换内存](https://www.elastic.co/guide/en/elasticsearch/reference/6.3/setup-configuration-memory.html)

> Swap 交换分区对性能和节点稳定性非常不利，一定要禁用。它会导致垃圾回收持续几分钟而不是几毫秒，并会导致节点响应缓慢，甚至与集群断开连接。



有三种方式可以实现 ES 不使用 Swap 分区

#### 2.3.1 Linux 系统中的关闭 Swap (临时有效)

执行命令

```
sudo swapoff -a
```

可以临时禁用 Swap 内存，但是操作系统重启后失效

#### 2.3.2 Linux 系统中的尽可能减少 Swap 的使用(永久有效)

执行下列命令

```
echo "vm.swappiness = 1">> /etc/sysctl.conf
```

正常情况下不会使用 Swap，除非紧急情况下才会 Swap。

#### 2.3.3 启用 bootstrap.memory_lock

config/elasticsearch.yml 文件增加配置

```
#锁定内存，不让 JVM 写入 Swap，避免降低 ES 的性能
bootstrap.memory_lock: true
```



### 2.4 减少分片数、副本数

**分片**

索引的大小取决于分片与段的大小，分片过小，可能导致段过小，进而导致开销增加；分片过大可能导致分片频繁 Merge，产生大量 IO 操作，影响写入性能。

因为我们每个索引的大小在 15G 以下，而默认是 5 个分片，没有必要这么多，所以调整为 3 个。

```
"index.number_of_shards": "3"
```

分片的设置我们也可以配置在索引模板。



**副本数**

减少集群副本分片数，过多副本会导致 ES 内部写扩大。副本数默认为 1，如果某索引所在的 1 个节点宕机，拥有副本的另一台机器拥有索引备份数据，可以让索引数据正常使用。但是数据写入副本会影响写入性能。对于日志数据，有 1 个副本即可。对于大数据量的索引，可以设置副本数为0，减少对性能的影响。

```
"index.number_of_replicas": "1"
```

分片的设置我们也可以配置在索引模板。




### 3.控制数据来源
#### 3.1 应用按规范打印日志

有的应用 1 天生成 10G 日志，而一般的应用只有几百到 1G。一天生成 10G 日志一般是因为部分应用日志使用不当，很多大数量的日志可以不打，比如大数据量的列表查询接口、报表数据、debug 级别日志等数据是不用上传到日志服务器，这些**即影响日志存储的性能，更影响应用自身性能**。



# 四、ES性能优化后的效果
**优化后的两周内ELK性能良好，没有使用上的问题：**

* ES 数据不再丢失

* 数据延时在 10 秒之内，一般在 5 秒可以查出

* 每个 ES 节点负载比较稳定，CPU 和内存使用率都不会过高，如下图

  ![ES节点运行情况.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6ea386a2a3f4265a5c1d29a9f823448~tplv-k3u1fbpfcp-zoom-1.image)


  欢迎关注公众号与笔者共同交流哦^_^

  ![](https://user-gold-cdn.xitu.io/2019/12/29/16f51ecd24e85b62?w=1002&h=270&f=jpeg&s=59118)