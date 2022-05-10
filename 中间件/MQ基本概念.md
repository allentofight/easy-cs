1. 幸福的烦恼

   张大胖最近是又喜又忧，喜的是业务量发展猛增，忧的是由于业务量猛增，一些原来不是问题的问题变成了大问题，比如说新会员注册吧，原来注册成功只要发个短信就行了，但随着业务的发展，现在注册成功也需要发 push，发优惠券,...等

   

   ![](https://img-blog.csdnimg.cn/img_convert/62c07dbd9a0108f3ed011d033bbdecd7.png)

   这样光注册用户这一步就需要调用很多服务，导致用户注册都需要花不少时间，假设每个服务调用需要 50 ms，那么光以上服务就需要调用 200 ms，而且后续产品还有可能再加一些`发新人红包`等活动，每加一个功能，除了引入额外的服务增加耗时外，还需要额外集成服务，重发代码，实在让人烦不胜烦，张大胖想一劳永逸地解决这个问题，于是找了 CTO Bill 来商量一下，看能否提供一些思路

   

   Bill 一眼就看出了问题的所在:你这个系统存在三个问题：`同步`,`耦合`,`流量暴增时系统被压垮的风险`

   * `同步`: 我们可以看到在注册用户后，需要同步调用其他模块后才能返回，**这是耗时高的根本原因！**
   * `耦合`:注册用户与其他模块严重耦合，体现在每调用一个模块，都需要在注册用户代码处集成其他模块的代码并重新发布，此时在这些流量中只有`注册用户`这一步是核心流程，其他都是次要流程，核心流程应该与次要流程解藕，否则只要其中一个次要流程调用失败，整个流程也就失败了，体现在前端就是明明已经注册成功了，但返回给用户的却是失败的
   * `流量暴增风险`：如果某天运营搞活动，比如注册后送新人红包，那么很有可能导致用户注册的流量暴增，那么由于我们的注册用户流程过长，很有可能导致注册用户的服务无法承载相应的流量压力而导致系统雪崩

   不愧是 CTO，一眼看出问题所在，「那该怎么解决呢」张大胖问到

   「大胖，你应该听说过一句话：任何软件问题都可以通过添加一层中间层来解决，如果不能，那就再加一层，同样的针对以上问题我们也可以添加一个中间层来解决，比如添加个队列，把用户注册这个事件放到队列中，让其他模块去这个队列里取这个事件然后再做相应的操作」Bill 边说边画出了他所说的中间层队列

   ![](https://img-blog.csdnimg.cn/img_convert/43eae0aa03a14b565cca7c6e1aa099e9.png)

   可以看到，这是个典型的生产者-消费者模型，用户注册后只要把注册事件丢给这个队列就可以立即返回，**实现了将同步变了异步**，其他服务只要从这个队列中拉取事件消费即可进行后续的操作，**同时也实现了注册用户逻辑与其他服务的解耦**，另外即使流量暴增也没有影响，因为注册用户将事件发给队列后马上返回了，这一发消息可能只要 5 ms，也就是说总耗时是 50ms+5ms = 55 ms，而原来的总耗时是 200 ms，系统的吞吐量和响应速度提升了近 4 倍，大大提升了系统的负责能力，这一步也就是我们常说的**削峰**，将暴增的流量放入队列中以实现平稳过渡

   

   「妙啊，加了一层队列就达到了**异步**，**解藕**，**削峰**的目的，也完美地解决了我的问题」张大胖兴奋地说

   

   「先别高兴得太早，你想想这个队列该用哪个，JDK 的内置队列是否可行，或者说什么样的队列才能满足我们的条件呢」Bill 提醒道

   张大胖想了一下如果直接使用 JDK 的队列（Queue）可能会有以下问题：

   1. 由于队列在生产者所在服务内存，其他消费者不得不从生产者中取，也就意味着生产者与消息者紧藕合，这显然不合理

   2. 消息丢失：现在是把消息存储在队列中，而队列是在内存中的，那如果机器宕机，队列中的消息不就丢失了吗，显然不可接受

   3. 单个队列中的消息只能被一个服务消费，也就是说如果某个服务从队列中取消息消费后，其他服务就取不了这个消息了,有一个办法倒是可以，为每一个服务准备一个队列，这样发送消息的时候只发送给一个队列，再通过这个队列把完整消息复制给其他队列即可![](https://img-blog.csdnimg.cn/img_convert/1809d2edd0d610d8ce526a037795685a.png)

      这种做法虽然理论上可以，但实践起来显然有问题，因为这就意味着每对接一个服务都要准备一份一模一样的队列，而且复制多份消息性能也存在严重问题，还得保证复制中消息不丢失，无疑增加了技术上的实现难度

   

   ### broker

   针对以上问题 Bill 和张大胖商量了一下决定自己设计一个独立于生产者和消费者的消息队列（姑且把中间这个保存消息的组件称为 Broker），这样的话就解决了问题一，生产者把消息发给 Broker，消费者只需把消息从 Broker 里拉出来消费即可，生产者和消费者就彻底解耦了，如下

   ![](https://img-blog.csdnimg.cn/img_convert/2e07f28442bc591087c1e7b20e1ce1e5.png)

   

   那么这个 Broker 应该如何设计才能满足我们的要求呢，显然它应该满足以下几个条件：

   1. **消息持久化**：不能因为 Broker 宕机了消息就都丢失了，所以消息不能只保存在内存中，应该持久化到磁盘上，比如保存在文件里，这样由于消息持久化了，它也可以被多个消费者消费，只要每个消费者保存相应的消费进度，即可实现多个消费者的独立消费
   2. **高可用**：如果 Broker 宕机了，producer 就发不了消息了，consumer 也无法消费，这显然是不可接受的，所以必须保证 Broker 的高可用
   3. **高性能**：我们定一个指标，比如 10w TPS，那么要实现这个目的就得满足以下三个条件：
      1. producer 发送消息要快（或者说 broker 接收消息要快）
      2. 持久化到文件要快
      3. consumer 拉取消息要快

   

   接下来我们再来看 broker 的整体设计情况

   

   针对问题一，我们可以把消息存储在文件中，消息通过**顺序写入文件**的方式来保证写入文件的高性能

   

   ![](https://img-blog.csdnimg.cn/img_convert/b0e2ccd3cca563141dc54323cd8730f6.png)

   

   顺序写文件的性能很高，接近于内存中的随机写，如下图示

   ![](https://img-blog.csdnimg.cn/img_convert/d3dc929207ee79e70ed59913791a93d9.png)

   这样 consumer 如果要消费的话，就可以从存储文件中读取消息了。好了，现在问题来了，我们都知道消息文件是存在硬盘中的，如果每次 broker 接收消息都写入文件，每次 consumer 读取消息都从硬盘读取文件，由于都是磁盘 IO，是非常耗时的，有什么办法可以解决呢

   ### page cache

   磁盘 IO 是很慢的，为了避免 CPU 每次读写文件都得和磁盘交互，一般先将文件读取到内存中，然后再由 CPU 访问，这样 CPU 直接在内存中读写文件就快多了，那么文件怎么从磁盘读取入内存呢，首先我们需要明白文件是以 block（块）的形式读取的，而 Linux 内核在内存中会以页大小（一般为 4KB）为分配单位。对文件进行读写操作时，内核会申请内存页（内存页即 page，多个 page 组成 page cache，即页缓存），然后将文件的 block 加载到页缓存中（n block size = 1 page size，如果一个 block 大小等于一个 page，则 n = 1）如下图示

   ![](https://img-blog.csdnimg.cn/img_convert/146a900746b03bd19b7dcb91e499b802.png)

   

   这样的话读写文件的过程就一目了解

   * **对于读文件**：CPU 读取文件时，首先会在 page cache 中查找是否有相应的文件数据，如果有直接对 page cache 进行操作，如果没有则会触发一个缺页异常（fault page）将磁盘上的块加载到 page cache 中，同时由于程序局部性原理，会一次性加载多个 page（读取数据所在的 page 及其相邻的 page ）到 page cache 中以保证读取效率
   * **对于写文件**：CPU 首先会将数据写入 page cache 中，然后再将 page cache 刷入磁盘中

   CPU 对文件的读写操作就转化成了对页缓存的读写操作，这样只要让 producer/consumer 在内存中读写消息文件，就避免了磁盘 IO

   

   ### mmap

   需要注意的是 page cache 是存在内核空间中的，还不能直接为应用程序所用，必须经由 CPU 将内核空间 page cache 拷贝到用户空间中才能为进程所用（同样的如果是写文件，也是先写到用户空间的缓冲区中，再拷贝到内核空间的 page cache，然后再刷盘）

   

   ![](https://img-blog.csdnimg.cn/img_convert/bb5dbb686605bae9fd9b33b7b9294bf6.png)

   **画外音**：为啥要将 page cache 拷贝到用户空间呢，这主要是因为页缓存处在内核空间，不能被用户进程直接寻址

   

   上图为程序读取文件完整流程：

   1. 首先是硬盘中的文件数据载入处于内核空间中的 page cache（也就是我们平常所说的内核缓冲区）
   2. CPU 将其拷贝到用户空间中的用户缓冲区中
   3. 程序通过用户空间的虚拟内存来映射操作用户缓冲区（两者通过 MMU 来转换），进而达到了在内存中读写文件的目的

   

   将以上流程简化如下

    

   ![](https://img-blog.csdnimg.cn/img_convert/479553bbb0998d8fb1d90371c9e7f9ee.png)

   以上是传统的文件读 IO 流程，可以看到程序的一次读文件经历了一次 read 系统调用和一次 CPU 拷贝，那么从内核缓冲区拷贝到用户缓冲区的这一步能否取消掉呢，答案是肯定的

   

   只要将虚拟内存映射到内核缓存区即可，如下

   ![mmap.drawio (2)](https://img-blog.csdnimg.cn/img_convert/3d3d5883cdd37c511e30afbfbd5fa298.png)

   

   可以看到使用这种方式有两个好处

   1. 省去了 CPU 拷贝，原本需要 CPU 从内核缓冲区拷贝到用户缓冲区，现在这一步省去了
   2. 节省了一半的空间: 因为不需要将 page cache 拷贝到用户空间了，可以认为用户空间和内核空间共享 page cache

   

   我们把这种通过将文件映射到进程的虚拟地址空间从而实现在内存中读写文件的方式称为 mmap（Memory Mapped Files）

   

   上面这张图画得有点简单了，再来看一下 mmap 的细节

   ![](https://img-blog.csdnimg.cn/img_convert/a008c5568fa38f60f2e5c89ece13d16d.png)

   

   

   

   1. 先把磁盘上的文件映射到进程的虚拟地址上（此时还未分配物理内存），即调用 mmap 函数返回指针 ptr，它指向虚拟内存中的一个地址，这样进程无需再调用 read 或 write 对文件进行读写，只需要通过 ptr 就能操作文件，所以如果需要对文件进行多次读写，显然使用 mmap 更高效，因为只会进行一次系统调用，比起多次 read 或 write 造成的多次系统调用显然开销会更低
   2. 但需要注意的是此时的 ptr 指向的是逻辑地址，并未真正分配物理内存，只有通过 ptr 对文件进行读写操作时才会分配物理内存，分配之后会更新页表，将虚拟内存与物理内存映射起来，这样虚拟内存即可通过 MMU 找到物理内存，分配完内存后即可将文件加载到 page cache，于是进程就可在内存中愉快地读写文件了

   

   使用 mmap 有力地提升了文件的读写性能，它也是我们常说的零拷贝的一种实现方式，既然 mmap 这么好，可能有人就要问了，那为什么文件读写不都用 mmap 呢，天下没有免费的午餐，mmap 也是有成本的，它有如下缺点

   1. **文件无法完成拓展**：因为执行 mmap 的时候，你所能操作的范围就已经确定了，无法增加文件长度

   2. **地址映射的开销**：为了创建并维持虚拟地址空间与文件的映射关系，内核中需要有特定的数据结构来实现这一映射。内核为每个进程维护一个任务结构 task_struct，task_struct 中的 mm_struct 描述了虚拟内存的信息，mm_struct 中的 mmap 字段是一个 vm_area_struct 指针，内核中的 vm_area_struct 对象被组织成一个链表 + 红黑树的结构。如下图示

      ![](https://img-blog.csdnimg.cn/img_convert/fe58fa043a5c911bcf955ebe2e2072d1.png)

      所以理论上，进程调用一次 mmap 就会产生一个 vm_area_struct 对象（不考虑内核自动合并相邻且符合条件的内存区域），vm_area_struct 数量的增加会增大内核的管理工作量，增大系统开销

   3. **缺页中断（page fault）的开销**: 调用 mmap 内核只是建立了逻辑地址（虚拟内存）到物理地址（物理内存）的映射表，实际并没有任何数据加载到物理内存中，只有在主动读写文件的时候发现数据所在分页不在内存中时才会触发缺页中断，分配物理内存，缺页中断一次读写只会触发一个 page 的加载，一个 page 只有 4k，想象一次，如果一个文件是 1G，那就得触发 256 次缺页中断！中断的开销是很大的，那么对于大文件来说，就会发生很多次的缺页中断，这显然是不可接受的，所以一般 mmap 得配合另一个系统调用 madvise，它有个**文件预热**的功能可以**建议**内核一次性将一大段文件数据读取入内存，这样就避免了多次的缺页中断，同时为了避免文件从内存中 swap 到磁盘，也可以对这块内存区域进行锁定，避免换出

   4. **mmap 并不适合读取超大型文件**，mmap 需要**预先分配连续的虚拟内存空间**用于映射文件，如果文件较大，对于 32 位地址空间（4 G）的系统来说，可能找不到足够大的连续区域，而且如果某个文件太大的话，会挤压其他热点小文件的 page cache 空间，影响这些文件的读写性能

   综上考虑，我们给每一个消息文件定为固定的 1G 大小，如果文件满了的话再创建一个即可，我们把这些存储消息的文件集合称为 **commitlog**。这样的设计还有另一个好处：在删除过期文件的时候会很方便，直接把之前的文件整个删掉即可，最新的文件无需改动，而如果把所有消息都写到一个文件里，显然删除之前的过期消息会非常麻烦

   ### consumeQueue 文件

   通过 mmap 的方式我们极大地提高了读写文件的效率，这样的话即可将 commitlog 采用 mmap 的方式加载到 page cache 中，然后再在 page cache 中读写消息，如果是写消息直接写入 page cache 当然没问题，但如果是读消息（消费者根据消费进度拉取消息）的话可就没这么简单了，当然如果每个消息的大小都一样，那么文件读取到内存中其实就相当于数组了，根据消息进度就能很快地定位到其在文件的位置（假设消息进度为 offset，每个消息的大小为 size，则所要消费的位置为 offset * size），但很显然每个消息的大小基本不可能相同，实际情况很可能是类似下面这样

   ![](https://img-blog.csdnimg.cn/img_convert/b316ee0c6c79d10f7a9b531135e93e9d.png)

   如图示，这里有三个消息，每个消息的消息体分别为 2kb，3kb，4kb，消息大小都不一样

   

   这样的话会有两个问题

   1. 消息边界不清，无法区分相邻的两个消息
   2. 即使解决了以上问题，也无法解决根据消费进度**快速定位**其所对应消息在文件的位置。假设 broker 重启了，然后读取消费进度（消费进度可以持久化到文件中），此时不得不从头读取文件来定位消息在文件的位置，这在效率上显然是不可接受的

   那能否既能利用到数组的快速寻址，又能快速定位消费进度对应消息在文件中的位置呢，答案是可以的，我们可以新建一个索引文件（我们将其称为 consumeQueue 文件），每次写入 commitlog 文件后，都把此消息在 commitlog 文件中的 offset（我们将其称为 commit offset，8 字节） 及其大小（size，4 字节）还有一个 tag hashcode（8 字节，它的作用后文会提到）这三个字段顺序写入 consumeQueue 文件中

   ![](https://img-blog.csdnimg.cn/img_convert/7ef68414420aebf8dfc954cc23fef189.png)

   

   ![](https://img-blog.csdnimg.cn/img_convert/e748b6988adbe1275d9f464e2672a9e7.png)

   

   

   这样每次追加写入 consumeQueue 文件的大小就固定为 20 字节了，由于大小固定，根据数组的特性，就能迅速定位消费进度在索引文件中的位置，然后即可获取 commitlog offset 和 size，进而快速定位其在 commitlog 中消息

   

   ![](https://tva1.sinaimg.cn/large/e6c9d24ely1h22j4cey97j20fi0b0wf0.jpg)

   这里有个问题，我们上文提到 commitlog 文件固定大小 1G，写满了会再新建一个文件，为了方便根据 commitlog offset 快速定位消息是在哪个 commitlog 的哪个位置，我们可以以消息偏移量来命名文件，比如第一个文件的偏移量是 0，第二个文件的偏移量为 1G（1024\*1024\*1024 = 1073741824 B），第三个文件偏移量为 2G（2147483648 B），如下图示

   ![](https://img-blog.csdnimg.cn/img_convert/ee7af1add27afc3e6ee4307b8b3dac2c.png)

   同理，consumeQueue 文件也会写满，写满后也要新建一个文件再写入,我们规定 consumeQueue 可以保存 30w 条数据，也就是 30w \* 20 byte = 600w Byte = 5.72 M，为了便于定位消费进度是在哪个 consumeQueue文件中，每个文件的名称也是以偏移量来命名的，如下

   ![](https://img-blog.csdnimg.cn/img_convert/a46df3810b59fadeb71afd21f0a8a2f4.png)

   知道了文件的写入与命名规则，我们再来看下消息的写入与消费过程

   1. 消息写入：首先是消息被顺序写入 commitlog 文件中，写入后此消息在文件中的偏移（commitlog offset）和大小（size）会被顺序写入相应的 consumeQueue 文件中
   2. 消费消息：每个消费者都有一个消费进度，由于每个 consumeQueue 文件是根据偏移量来命名的，首先消费进度可根据二分查找快速定位到进度是在哪个 consumeQueue 文件，进一步定义到是在此文件的哪个位置，由此可以读取到消息的 commitlog offset 和 size，然后由于 commitlog 每个文件的命名都是按照偏移量命名的，那么根据 commitlog offset 显然可以根据二分查找快速定位到消息是在哪个 commitlog 文件，进而再获取到消息在文件中的具体位置从而读到消息

   同样的为了提升性能， consumeQueue 也利用了 mmap 进行读写

   有人可能会说这样查找了两次文件，性能可能会有些问题，实际上并不会，根据前文所述，可以使用 mmap + 文件预热 + 锁定内存来将文件加载并一直保留到内存中，这样不管是 commitlog 还是 consumeQueue 都是在 page cache 中的，既然是在内存中查找文件那性能就不是问题了

   

   ### 对 ConsumeQueue 的改进--数据分片

   目前为止我们讨论的场景是多个消费者独立消费消息的场景，这种场景我们将其称为`广播模式`，这种情况下每个消费者都会全量消费消息，但还有一种更常见的场景我们还没考虑到，那就是`集群模式`，集群模式下每个消费者只会消费**部分消息**，如下图示：

   ![](https://img-blog.csdnimg.cn/img_convert/5e2df3919bf63c391c86a3c077828fb2.png)

   

   **集群模式下每个消费者采用负载均衡的方式分别并行消费一部分消息，主要目的是为了加速消息消费以避免消息积压**，那么现在问题来了，Broker 中只有一个 consumerQueue，显然没法满足集群模式下并行消费的需求，该怎么办呢，我们可以借鉴分库分表的设计理念：**将数据分片存储**，具体做法是创建多个 consumeQueue，然后将数据平均分配到这些 consumerQueue 中，这样的话每个 consumer 各自负责独立的 consumerQueue 即可做到并行消费

   ![](https://img-blog.csdnimg.cn/img_convert/45d9a32c3678a68fddff09ec1f7d76e8.png)

   

   如图示: Producer 把消息负载均衡分别发送到 queue 0 和 queue 1 队列中，consumer A 负责 queue 0，consumer B 负责 queue 1 中的消息消费，这样可以做到并行消费，极大地提升了性能

   ### topic

   现在所有消息都持久化到 Broker 的文件中，都能被 consumer 消费了，但实际上某些 consumer 可能只对某一类型的消息感兴趣，比如只对订单类的消息感兴趣，而对用户注册类的消息无感，那么现在的设计显然不合理，所以需要对消息进行进一步的细分，我们把**同一种业务类型的的消息集合称为 Topic**。这样消费者就可以只订阅它感兴趣的 Topic 进行消费，因此也不难理解 consumeQueue 是针对 Topic 而言的，producer 发送消息时都会指定消息的 Topic，消息到达 Broker 后会发送到 Topic 中对应的 consumeQueue，这样消费者就可以只消费它感兴趣的消息了

   ![](https://img-blog.csdnimg.cn/img_convert/10ce8d74e22a822ea2ad0edd5b39e5c2.png)

   

   ### tag

   把消息按业务类型划分成 Topic 粒度还是有点大，以订单消息为例，订单有很多种状态，比如`订单创建`，`订单关闭`,`订单完结`等，某些消费者可能只对某些订单状态感兴趣，所以我们有时还需要进一步对某个 Topic 下的消息进行分类，我们将这些分类称为 **tag**，比如订单消息可以进一步划分为`订单创建`，`订单关闭`,`订单完结`等 tag

   ![topic 与 tag 关系](https://img-blog.csdnimg.cn/img_convert/b723de22483990e55c599e936f0cabfa.png)

   

   producer 在发消息的时候会指定 topic 和 tag，Broker 也会把 topic, tag 持久化到文件中，那么 consumer 就可以只订阅它感兴趣的 topic + tag 消息了，现在问题来了，consumer 来拉消息的时候，Broker 怎么只传给 consumer 根据 topic + tag 订阅的消息呢

   

   还记得上文中提到消息持久化到 commitlog 后写入 consumeQueue 的信息吗


   ![](https://img-blog.csdnimg.cn/img_convert/941f97e61bd4d6b8f0f6bcf1e095d4aa.png)


   主要写入三个字段，最后一个字段为 tag 的 hashcode，这样的话由于 consumer 在拉消息的时候会把 topic，tag 发给 Broker ，Broker 就可以先根据 tag 的 hashcode 来对比一下看看此消息是否符合条件，如果不是略过继续往后取，如果是再从 commitlog 中取消息后传给 consumer，有人可能会问为什么存的是 tag hashcode 而不是 tag，主要有两个原因

   1. hashcode 是整数，整数对比更快
   2. 为了保证此字段为固定的字节大小（hashcode 为 int 型，固定为 4 个字节），这样每次写入 consumeQueue 的三个字段即为固定的 20 字节，即可利用数组的特性快速定位消息进度在文件中的位置，如果用 tag 的话，由于 tag 是字符串，是变长的，没法保证固定的字节大小

   

   至此我们简单总结下消息的发送，存储与消息流程

   ![](https://img-blog.csdnimg.cn/img_convert/8187f95618596ea34ef2f1602c03b06e.png)

   

   1. 首先 producer 发送 topic，queueId，message 到 Broker 中，Broker 将消息通过顺序写的形式持久化到 commitlog 中，这里的 queueId 是 Topic 中指定的 consumeQueue 0，consumeQueue 1，consumeQueue ...，一般通过负载均衡的方式轮询写入对应的队列，比如当前消息写入 consumeQueue 0，下一条写入 consumeQueue 1,...，不断地循环
   2. 持久化之后可以知道消息在 commitlog 文件中的偏移量和消息体大小，如果 consumer 指定订阅了 topic 和 tag，还会算出 tag hashCode，这样的话就可以将这三者顺序写入 queueId 对应的 consumeQueue 中
   3. 消费者消费：每一个 consumeQueue 都能找到每个消费者的消息进度（consumeOffset），据此可以快速定位其所在的 consumeQueue 的文件位置，取出 commitlog offset，size，tag hashcode 这三个值，然后首先根据 tag hashcode 来过滤消息，如果匹配上了再根据 commitlog offset，size 这两个元素到 commitlog 中去查找相应的消息然后再发给消费者

   **注意**：所有 Topic 的消息都写入同一个 commitlog 文件（而不是每个 Topic 对应一个 commitlog 文件），然后消息写入后会根据 topic,queueId 找到 Topic 所在的 consumeQueue 再写入

   

   

   需要注意的是我们的 Broker 是要设定为高性能的（10 w QPS）那么上面这些步骤有两个瓶颈点

   1. producer 发送消息到持久化至 commitlog 文件的性能问题

      ![](https://img-blog.csdnimg.cn/img_convert/7895c81eeb05d67db6a3152c222c5ffa.png)

      

      如图示，Broker 收到消息后是先将消息写到了内核缓冲区 的 page cache 中，最终将消息刷盘，那么消息是写到 page cache 返回 ack，还是刷盘后再返回呢，这取决于你消息的重要性，如果是像日志这样的消息，丢了其实也没啥影响，这种情况下显然可以选择写到 page cache 后就马上返回，OS 会择机将其刷盘，这种刷盘方式我们将其称为**异步刷盘**，这也是大多数业务场景选择的刷盘方式，这种方式其实已经足够安全了，哪怕 JVM 挂掉了，由于 page cache 是由 OS 管理的，OS 也能保证将其刷盘成功，除非 Broker 机器宕机。当然对于像转账等安全性极高的金融场景，我们可能还是要将消息从 page cache 刷盘后再返回 ack，这种方式我们称为**同步刷盘**，显然这种方式会让性能大大降低，使用要慎重

   2. consumer 拉取消息的性能问题

      很显然这一点不是什么问题，上文提到，不管是 commitlog 还是 consumeQueue 文件，都缓存在 page cache 中，那么直接从 page cache 中读消息即可，由于是基于内存的操作，不存在什么瓶颈，当然这是基于消费进度与生产进度差不多的前提，如果某个消费者指定要从某个进度开始消费，且此进度对应的 commitlog 文件不在 page cache 中，那就会触发磁盘 IO

      

      

   ### Broker 的高可用

    上文我们都是基于一个 Broker 来讨论的，这显然有问题，Broker 如果挂了，依赖它的 producer，consumer 不就也嗝屁了吗，所以 broker 的高可用是必须的，一般采用主从模式来实现 broker 的高可用

   ![](https://img-blog.csdnimg.cn/img_convert/8f767609747475dfcd34a13d8040787d.png)

   如图示：Producer 将消息发给 主 Broker ，然后 consumer 从主 Broker 里拉消息，而 从 Broker 则会从主 Broker 同步消息，这样的话一旦主 Broker 宕机了，consumer 可以从 Broker 里拉消息，同时在 RocketMQ 4.5 以后，引入一种 dledger 模式，这种模式要求一主多从（至少 3 个节点），这样如果主 Broker 宕机后，另外多个从 Broker 会根据 Raft 协议选举出一个主 Broker，Producer 就可以向这个新选举出来的主节点发送消息了

   

   如果 QPS 很高只有一个主 Broker 的话也存在性能上的瓶颈，所以生产上一般采用多主的形式，如下图示

   ![](https://img-blog.csdnimg.cn/img_convert/2f7028d5fc3c6583d0852fbc8f4290fc.png)

   这样的话 Producer 可以负载均衡地将消息发送到多个 Broker 上，提高了系统的负载能力，不难发现这意味着 Topic 是分布式存储在多个 Broker 上的，而 Topic 在每个 Broker 上的存储都是以多个 consumeQueue 的形式存在的，这极大地提升了 Topic 的水平扩展与系统的并发执行能力

   ![](https://img-blog.csdnimg.cn/img_convert/c784488d81a508c165046ce22cf74d40.png)

   

   

   

   ### nameserver

   目前为止我们的设计貌似不错，通过一系列设计让 Broker 满足了高性能，高扩展的要求，但我们似乎忽略了一个问题，Producer，Consumer 该怎么和 Broker 通信呢，一种做法是在 Producer，Consumer 写死要通信的 Broker ip 地址，虽然可行，但这么做的话显然会有很大的问题，配置死板，扩展性差，考虑以下场景

   1. 如果扩容（新增 Broker)，producer 和 consumer 是不是也要跟着新增 Broker ip 地址
   2. 每次新增 Topic 都要指定在哪些 Broker 存储，我们知道 producer 在发消息，consumer 在订阅消息的时候都要指定对应的 Topic ，那就意味着每次新增 Topic 后都需要在 producer，consumer 做相应变更（记录 topic -> broker 地址）
   3. 如果 broker 宕机了，producer 和 consumer 需要将其从配置中移除，这就意味着 producer,consumer 需要与相关的 brokers 通过心跳来通信以便知道其存活与否，这样无疑增加了设计的复杂度

   参考下 dubbo 这类 RPC 框架，你会发现基本上都会新增一个类似 Zookeeper 这样的`注册中心`的中间层（一般称其为 nameserver），如下

   

   ![](https://img-blog.csdnimg.cn/img_convert/673e2812a803354f8e5d2433e378d5c2.png)

   主要原理如下：

   为了保证高可用，一般 nameserver 以集群的形式存在（至少两个），Broker 启动后不管主从都会向每一个 nameserver 注册，注册的信息有哪些呢，想想看 producer 要发消息给 broker 需要知道哪些信息呢，首先发消息要指定 Topic，然后要指定 Topic 所在的 broker，再然后是知道 Topic 在 Broker 中的队列数量（可以这样负载均衡地将消息发送到这些 queue 中），所以 broker 向 nameserver 注册的信息中应该包含以下信息

   ![page_cache.drawio (1)](https://img-blog.csdnimg.cn/img_convert/a541b2fcbf321d59ee1c7cc57c4f3e2c.png)

   

   这样的话 producer 和 consumer 就可以通过与 nameserver 建立长连接来定时（比如每隔 30 s）拉取这些路由信息从而更新到本地，发送/消费消息的时候就可以依据这些路由信息进行发送/消费

   

   那么加了一个 nameserver 和原来的方案相比有什么好处呢，可以很明显地看出：producer/consumer 与具体的 broker 解藕了，极大提升了整体架构的可扩展性：

   1. producer/consumer 的所有路由信息都能通过 nameserver 得到，比如现在要在 brokers 上新建一个 Topic，那么 brokers 会把这些信息同步到 nameserver，而 producer/consumer 会定时去 nameserver 拉取这些路由信息更新到本地，做到了路由信息配置的自动化
   2. 同样的如果某些 broker 宕机了，由于 broker 会定时上报心跳到 nameserver 以告知其存活状态，一旦 nameserver 监测到 broker 失效了，producer/consumer 也能从中得到其失效信息，从而在本地路由中将其剔除

   

   可以看到通过加了一层 nameserver，producer/consumer 路由信息做到了配置自动化，再也不用手动去操作了，整体架构甚为合理

   

   

   ### 总结

   以上即我们所要阐述的 RocketMQ 的设计理念，基本上涵盖了重要概念的介绍，我们再来简单回顾一下:

   

   首先根据业务场景我们提出了 RocketMQ 设计的三大目标:消息持久化，高性能，高可用，毫无疑问 broker 的设计是实现这三大目标的关键，为了消息持久化，我们设计了 commitlog 文件，通过顺序写的方式保证了文件写入的高性能，但如果每次 producer 写入消息或者 consumer 读取消息都从文件来读写，由于涉及到磁盘 IO 显然性能会有很大的问题，于是我们了解到操作系统读写文件会先将文件加载到内存中的 page cache 中。对于传统的文件 IO，由于 page cache 存在内核空间中，还需要将其拷贝到用户空间中才能为进程所用（同样的，写入消息也要写将消息写入用户空间的 buffer，再拷贝到 内核空间中的 page cache），于是我们使用了 mmap 来避免了这次拷贝，这样的话 producer 发送消息只要先把消息写入 page cache 再异步刷盘，而 consumer 只要保证消息进度能跟得上 producer 产生消息的进度，就可以直接从 page cache 中读取消息进行消费，于是 producer 与 consumer 都可以直接从 page cache 中读写消息，极大地提升了消息的读写性能，那怎么保证 consumer 消费足够快以跟上 producer 产生消息的速度的，显然，让消息分布式，分片存储是一种通用方案，这样的话通过增加 consumer 即可达到并发消费消息的目的

   

   最后，为了避免每次创建 Topic 或者 broker 宕机都得修改 producer/consumer 上的配置，我们引入了 nameserver， 实现了服务的自动发现功能。

   

   仔细与其它 RPC 框架横向对比后，你会发现这些 RPC 框架用的思想其实都很类似，比如数据使用分片存储以提升数据存储的水平扩展与并发执行能力，使用 zookeeper，nameserver 等注册中心来达到服务注册与自动发现的目的，所以掌握了这些思想， 我们再去观察学习或设计 RPC 时就能达到事半功倍的效果

   

      

      

      

      

      

      
