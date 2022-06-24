上周[线程崩溃为什么不会导致 JVM 崩溃](https://mp.weixin.qq.com/s/JnlTdUk8Jvao8L6FAtKqhQ)在其他平台发出后，有一位小伙伴留言说有个地方不严谨

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h3gmf7nigqj20hr03cwen.jpg)

他认为如果 JVM 中的主线程异常没有被捕获，JVM 还是会崩溃，那么这个说法是否正确呢，我们做个试验看看结果是否是他说的这样

```java
public class Test {
    public static void main(String[] args) {
        TestThread testThread = new TestThread();
        TestThread.start();
        Integer p = null;
      	// 这里会导致空指针异常
        if (p.equals(2)) {
            System.out.println("hahaha");
        }
    }
}

class TestThread extends Thread {
    @Override
    public void run()  {
        while (true) {
            System.out.println("test");
        }
    }
}
```

试验很简单，首先启动一个线程，在这个线程里搞一个 while true 不断打印， 然后在主线程中制造一个空指针异常，不捕获，然后看是否会一直打印 test



结果是会不断打印 test，说明**主线程崩溃，JVM 并没有崩溃**，这是怎么回事， JVM 又会在什么情况下完全退出呢？



其实在 Java 中并没有所谓主线程的概念，只是我们习惯把启动的线程作为主线程而已，所有线程其实都是平等的，不管什么线程崩溃都不会影响到其它线程的执行，注意我们这里说的线程崩溃是指由于未 catch 住 JVM 抛出的虚拟机错误（VirtualMachineError）而导致的崩溃，虚拟机错误包括 InternalError，OutOfMemoryError，StackOverflowError，UnknownError 这四大子类

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h3il0s8j2vj21660hst9n.jpg)



JVM 抛出这些错误其实是一种防止整个进程崩溃的自我防护机制，这些错误其实是 JVM 内部定义了信号处理函数处理后抛出的，JVM 认为这些错误"罪不致死"，所以选择恢复线程再给这些线程抛错误（就算线程不 catch 这些错误也不会崩溃）的方式来避免自身崩溃，但如果线程触发了一些其他的非法访问内存的错误，JVM 则会认为这些错误很严重，从而选择退出，比如下面这种非法访问内存的错误就会被认为是致命错误，JVM 就不会向上层抛错误，而会直接选择退出

```java
Field f = Unsafe.class.getDeclaredField("theUnsafe");
f.setAccessible(true);
Unsafe unsafe = (Unsafe) f.get(null);
unsafe.putAddress(0, 0);
```

回过头来看，除了这些致命性错误导致的 JVM 崩溃，还有哪些情况会导致 JVM 退出呢，在 javadoc 上说的很清楚

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h3gn58lj6hj20it08xjs8.jpg)

**The Java Virtual Machine exits when the only threads running are all daemon threads**

也就是说只有在 JVM 的所有线程都是守护线程（daemon thread）的时候才会完全退出，什么是守护线程？守护线程其实是为其他线程服务的线程，比如垃圾回收线程就是典型的守护线程，既然是为其他线程服务的，那么一旦其他线程都不存在了，守护线程也没有存在的意义了，于是 JVM 也就退出了，守护线程通常是 JVM 运行时帮我们创建好的，当然我们也可以自己设置，以开头的代码为例，在创建完 TestThread 后，调用 testThread.setDaemon(true) 方法即可将线程转为守护线程，然后再启动，这样在主线程退出后，JVM 就会退出了，大家可以试试



### Java 线程模型简介

我们可以看看 Java 的线程模型，这样大家对 JVM 的线程调度也会有一个更全面的认识，我们可以先从源码角度看看，启动一个 Thread 到底在 JVM 内部发生了什么，启动源码代码在 Thread#start 方法中

```java
public class Thread {
  
  public synchronized void start() {
		...
    start0();
    ...
  }
  private native void start0();
}
```

可以看到最终会调用 start0 这个 native 方法，我们去下载一下 openJDK（地址：https://github.com/AdoptOpenJDK/openjdk-jdk8u） 来看看这个方法对应的逻辑



![image-20220622073357619](https://tva1.sinaimg.cn/large/e6c9d24ely1h3go9alx08j20jb079wfb.jpg)

可以看到 start0 对应的是 JVM_startThread 这个方法，我们主要观察在 Linux 下的线程启动情况，一路追踪下去

```c++
// jvm.cpp
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  native_thread = new JavaThread(&thread_entry, sz);

// thread.cpp
JavaThread::JavaThread(ThreadFunction entry_point, size_t stack_sz)
{
  os::create_thread(this, thr_type, stack_sz);
}

// os_linux.cpp
bool os::create_thread(Thread* thread, ThreadType thr_type, size_t stack_size) {
  int ret = pthread_create(&tid, &attr, (void* (*)(void*)) java_start, thread);
}
```

可以看到最终是通过调用 pthread_create 来启动线程的，这个方法是一个 C 函数库实现的创建 native thread 的接口，是一个系统调用，由此可见 pthread_create 最终会创建一个 native thread，这个线程也叫**内核线程**，操作系统只能调度内核线程，于是我们知道了在 Java 中，Java 线程和内核线程是一对一的关系，Java 线程调度实际上是通过操作系统调度实现的，这种一对一的线程也叫 NPTL（Native POSIX Thread Library） 模型，如下

![NPTL线程模型](https://tva1.sinaimg.cn/large/e6c9d24ely1h3gx924x17j20qq0mqq5a.jpg)

那么这个内核线程在内核中又是怎么表示的呢， 其实在 Linux 中不管是进程还是线程都是通过一个 task_struct 的结构体来表示的， 这个结构体定义了进程需要的虚拟地址，文件描述符，寄存器，信号等资源



早期没有线程的概念，所以每次启动一个进程都需要调用 fork 创建进程，这个 fork 干的事其实就是 copy 父进程对应的 task_struct 的多数字段（pid 等除外），这在性能上显然是无法接受的。于是线程的概念被提出来了，线程除了有自己的栈和寄存器外，其他像虚拟地址，文件描述符等资源都可以共享



![线程共享代码段，数据段，地址空间，文件等资源](https://tva1.sinaimg.cn/large/e6c9d24ely1h3helilwsjj20en0e8q3c.jpg)



于是针对线程，我们就可以指定在创建 task_struct 时，采用**共享**而不是复制字段的方式。其实不管是创建进程（fork）还是创建线程（pthread_create）最终都会通过调用 clone() 的形式来创建 task_struct，只不过 pthread_create 在调用 clone 时，指定了如下几个共享参数

```
clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND, 0);
```

**画外音**：CLONE_VM 共享页表，CLONE_FS 共享文件系统信息，CLONE_FILES 共享文件句柄，CLONE_SIGHAND 共享信号



通过共享而不是复制资源的形式极大地加快了线程的创建，另外线程的调度开销也会更小，比如在（同一进程内）线程间切换的时候由于共享了虚拟地址空间，TLB 不会被刷新从而导致内存访问低效的问题



提到这相信你已经明白了教科书上的一句话：进程是资源分配的最小单元，而线程是程序执行的最小单位。在 Linux 中进程分配资源后，线程通过共享资源的方式来被调度的以提升线程的执行效率



由此可见，在 Linux 中所有的进程/线程都是用的 task_struct，**它们之间其实是平等的**，那怎么表示这些线程属于同一个进程的概念呢，毕竟线程之间也是要通信的，一组线程以及它们所共同引用的一组资源就是一个进程。, 它们还必须被视为一个整体。



task_struct 中引入了线程组的概念，如果线程都是由同一个进程（即我们说的主线程）产生的， 那么它们的 tgid（线程组id） 是一样的，如果是主线程，则 pid = tgid，如果是主线程创建的线程，则这些线程的 tgid 会与主线程的 tgid 一致，



那么在 LInux 中进程，进程内的线程之间是如何通信或者管理的呢，其实 NPTL 是一种实现了 POSIX Thread 的标准 ，所以我们只需要看 POSIX Thread 的标准即可，以下列出了 POSIX Thread 的主要标准：

1. 查看进程列表的时候, 相关的一组 task_struct 应当被展现为列表中的一个节点（即进程内如果有多个线程，展示进程列表 `ps -ef` 时只会展示主线程，如果要查看线程的话可以用 `ps -T`）
2. 发送给这个进程的信号(对应 kill 系统调用), 将被对应的这一组 task_struct 所共享, 并且被其中的任意一个”线程”处理
3. 发送给某个线程的信号(对应 pthread_kill), 将只被对应的一个 task_struct 接收, 并且由它自己来处理
4. 当进程被停止或继续时(对应 SIGSTOP/SIGCONT 信号), 对应的这一组 task_struct 状态将改变
5. 当进程收到一个致命信号(比如由于段错误收到 SIGSEGV 信号), 对应的这一组 task_struct 将全部退出

**画外音**: POSIX 即可移植操作系统接口（Portable Operating System Interface of UNIX，缩写为 POSIX ），是一种接口规范，如果系统都遵循这个标准，可以做到源码级的迁移，这就类似 Java 中的针对接口编程



这样就能很好地满足进程退出线程也退出，或者线程间通信等要求了



### NPTL 模型的缺点

 NPTL 是一种非常高效的模型，研究表明 NPTL 能够成功地在 IA-32 平台上在两秒种内生成 100,000 个线程，而 2.6 之前未采用 NPTL 的内核则需耗费 15 分钟左右，看起来 NPTL 确实很好地满足了我们的需求，但针对内核线程来调度其实还是有以下问题



1. 不管是进程还是线程，每次阻塞、切换都需要陷入系统调用(system call)，系统调用开销其实挺大的，包括上下文切换（寄存器切换），特权模式切换等，而且还得先让 CPU 跑操作系统的调度程序，然后再由调度程序决定该跑哪一个进程(线程)
2. 不管是进程还是线程，都属于抢占式调度（高优先级线进程优先被调度），由于抢占式调度执行顺序无法确定的特点，使用线程时需要非常小心地处理同步问题
3. 线程虽然更轻量级，但这只是相对于进程而言，实际上使用线程所消耗的资源依然很大，比如在 linux 上，一个线程默认的栈大小是1M，创建几万个线程就吃不消了



### 协程

NPTL 模型其实已经足够优秀了，上述问题本质上其实还是因为线程还是太“重”所致，那能否再在线程上抽出一个更轻量级的执行单元（可被 CPU 调度和分派的基本单位）呢，答案是肯定的，在线程之上我们可以再抽象出一个协程（coroutine）的概念,就像进程是由线程来调度的，同样线程也可以细化成一个个的协程来调度



![](https://tva1.sinaimg.cn/large/e6c9d24ely1h3hub066gyj20mq0ge0tq.jpg)



针对以上问题，协程都做了非常好的处理

1. 协程的调度处于用户态，也就没有了系统调用这些开销
2. 协程不属于抢占式调度，而是协作式调度，如何调度，在什么时间让出执行权给其它协程是由用户自己决定的，这样的话同步的问题也基本不存在，可以认为协程是无锁的，所以性能很高
3. 我们可以认为线程的执行是由一个个协程组成的，协程是更轻量的存在，内存使用大约只有线程的十分之一甚至是几十分之一，它是使用栈内存按需使用的，所以创建百万级的协程是非常轻松的事



协程是怎么做到上述这些的呢



协程（coroutine）可以分为两个角度来看，一个是 routine 即执行单元，一个是 co 即 cooperative 协作，也就是说线程可以依次顺序执行各个协程，但协程与线程不同之处在于，如果某个协程（假设为 A）内碰到了 IO 等阻塞事件，可以主动让出自己的调度权，即挂起（suspend），转而执行其他协程，等 IO 事件准备好了，再来调度协程 A

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h3ij7az1t2j20sg0dgdgd.jpg)



这就好比我在看电视的时候碰到广告，那我可以先去倒杯水，等广告播完了再回来继续看电视。而如果是函数，那你必须看完广告再去倒水，显然协程的效率更高。那么协程之间是怎么协作的呢，我们可以在两个协程之间碰到 IO 等阻塞事件时随时将自己挂起（yield），然后唤醒（resume）对方以让对方执行，想象一下如果协程中有挺多 IO 等阻塞事件时，那这种协作调度是非常方便的

![两个协程之间的“协作”](https://tva1.sinaimg.cn/large/e6c9d24ely1h3i37qdzv7j208b061dg0.jpg)

不像函数必须执行完才能返回，协程可以在执行流中的**任意位置**由用户决定挂起和唤醒，无疑协程是更方便的

![函数与协程的区别](https://tva1.sinaimg.cn/large/e6c9d24ely1h3ihyl8p7ij20ye0ig0ul.jpg)



更重要的一点是不像线程的挂起和唤醒等调度必须通过系统调用来让内核调度器来调度，**协程的挂起和唤醒完全是由用户决定的**，而且这个调度是在用户态，几乎没有开销！



前面我们一直提到一般我们在协程中碰到 IO 等阻塞事件时才会挂起并唤醒其他协程，所以可知**协程非常适合 IO 密集型的应用**，如果是计算密集型其实用线程反而更加合适



为什么 Go 语言这么最近这么火，一个很重要的原因就是因为因为它天生支持协程，可以轻而易举地创建成千上万个协程，而如果是创建线程的话，创建几百个估计就够呛了，不过比较遗憾的是 Java 原生并不支持协程，只能通过一些第三方库如 Quasar 来实现，2018 年 OpenJDK 官方创建了一个  loom 项目来推进协程的官方支持工作



### 总结

从进程，到线程再到协程，可知我们一直在想办法让执行单元变得更轻量级，一开始只有进程的概念，但是进程的创建在 Linux 下需要调用 fork 全部复制一遍资源，虽然后来引入了写时复制的概念，但进程的创建开销依然很大，于是提出了更轻量级的线程，在 Linux 中线程与进程其实都是用 task_struct 表示的，只是线程采用了共享资源的方式来创建，极大了提升了 task_struct 的创建与调度效率，但人们发现，线程的阻塞，唤醒都要通过系统调用陷入内核态才能被调度程度调度，如果线程频繁切换，开销无疑是很大的，于是人们提出了协程的概念，协程是根据栈内存按需求分配的，所需开销是线程的几十分之一，非常的轻量，而且调度是在用户态，并且它是协作式调度，可以很方便的挂起恢复其他协程的执行，在此期间，线程是不会被挂起的，所以无论是创建还是调度开销都很小，目前 Java 官方还不支持，不过支持协程应该是大势所趋，未来我们可以期待一下
