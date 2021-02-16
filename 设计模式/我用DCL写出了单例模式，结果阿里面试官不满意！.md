>  本文已收录到我的 github 地址: https://github.com/allentofight/easy-cs，欢迎大家关注并给个 star，这对我非常重要，感谢支持！之后码海的每篇文章都会收录至此地址以方便大家查阅!

## 前言
单例模式可以说是设计模式中最简单和最基础的一种设计模式了，哪怕是一个初级开发，在被问到使用过哪些设计模式的时候，估计多数会说单例模式。但是你认为这么基本的”单例模式“真的就那么简单吗？或许你会反问：「一个简单的单例模式该是咋样的？」哈哈，话不多说，让我们一起拭目以待，**坚持看完，相信你一定会有收获！**

## 饿汉式

饿汉式是最常见的也是最不需要考虑太多的单例模式，因为他不存在线程安全问题，饿汉式也就是在类被加载的时候就创建实例对象。饿汉式的写法如下：

```java
public class SingletonHungry {
    private static SingletonHungry instance = new SingletonHungry();

    private SingletonHungry() {
    }

    private static SingletonHungry getInstance() {
        return instance;
    }
}
```

- 测试代码如下：

```java
class A {
    public static void main(String[] args) {
        IntStream.rangeClosed(1, 5)
                .forEach(i -> {
                    new Thread(
                            () -> {
                                SingletonHungry instance = SingletonHungry.getInstance();
                                System.out.println("instance = " + instance);
                            }
                    ).start();
                });
    }
}

```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7d8185d8d844365be732fc78485e84e~tplv-k3u1fbpfcp-zoom-1.image)



优点:线程安全，不需要关心并发问题，写法也是最简单的。

缺点:在类被加载的时候对象就会被创建，也就是说不管你是不是用到该对象，此对象都会被创建，浪费内存空间



## 懒汉式

以下是最基本的饿汉式的写法，在单线程情况下，这种方式是非常完美的，但是我们实际程序执行基本都不可能是单线程的，所以这种写法必定会存在线程安全问题

```java
public class SingletonLazy {
    private SingletonLazy() {
    }

    private static SingletonLazy instance = null;

    public static SingletonLazy getInstance() {
        if (null == instance) {
            return new SingletonLazy();
        }
        return instance;

    }
}
```

演示多线程执行

```java
class B {
    public static void main(String[] args) {
        IntStream.rangeClosed(1, 5)
                .forEach(i -> {
                    new Thread(
                            () -> {
                                SingletonLazy instance = SingletonLazy.getInstance();
                                System.out.println("instance = " + instance);
                            }
                    ).start();
                });
    }
}

```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d633e9395cb84eca935cc69313c27730~tplv-k3u1fbpfcp-zoom-1.image)

结果很显然，获取的实例对象不是单例的。也就是说这种写法不是线程安全的，也就不能在多线程情况下使用

## DCL（双重检查锁式）

 DCL 即 Double Check Lock 就是在创建实例的时候进行双重检查，首先检查实例对象是否为空，如果不为空将当前类上锁，然后再判断一次该实例是否为空，如果仍然为空就创建该是实例；代码如下：

```java
public class SingleTonDcl {
    private SingleTonDcl() {
    }

    private static SingleTonDcl instance = null;

    public static SingleTonDcl getInstance() {
        if (null == instance) {
            synchronized (SingleTonDcl.class) {
                if (null == instance) {
                    instance = new SingleTonDcl();
                }
            }
        }
        return instance;
    }
}
```

测试代码如下：

```java
class C {
    public static void main(String[] args) {
        IntStream.rangeClosed(1, 5)
                .forEach(i -> {
                    new Thread(
                            () -> {
                                SingleTonDcl instance = SingleTonDcl.getInstance();
                                System.out.println("instance = " + instance);
                            }
                    ).start();
                });
    }
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/905097e5b3844f848961683a8c6c221c~tplv-k3u1fbpfcp-zoom-1.image)

相信大多数初学者在接触到这种写法的时候已经感觉是「高大上」了，首先是判断实例对象是否为空，如果为空那么就将该对象的 Class 作为锁，这样保证同一时刻只能有一个线程进行访问，然后再次判断实例对象是否为空，最后才会真正的去初始化创建该实例对象。一切看起来似乎已经没有破绽，但是当你学过JVM后你可能就会一眼看出猫腻了。没错，问题就在 instance = new SingleTonDcl(); 因为这不是一个原子的操作，这句话的执行是在 JVM 层面分以下三步：

1.给 SingleTonDcl 分配内存空间
2.初始化 SingleTonDcl 实例
3.将 instance 对象指向分配的内存空间（ instance 为 null 了）

正常情况下上面三步是顺序执行的，但是实际上JVM可能会「自作多情」得将我们的代码进行优化，可能执行的顺序是1、3、2，如下代码所示

```java
public static SingleTonDcl getInstance() {
    if (null == instance) {
        synchronized (SingleTonDcl.class) {
            if (null == instance) {
                1. 给 SingleTonDcl 分配内存空间
                3.将 instance 对象指向分配的内存空间（ instance 不为 null 了）
                2. 初始化 SingleTonDcl 实例
            }
        }
    }
    return instance;
}
```

假设现在有两个线程 t1, t2

1. 如果 t1 执行到以上步骤 3 被挂起
2. 然后 t2 进入了 getInstance 方法，由于 t1 执行了步骤  3，此时的 instance 已经不为空了，所以 if (null == instance) 这个条件不为空，直接返回 instance, 但由于 t1 还未执行步骤 2，导致此时的 instance 实际上是个半成品，会导致不可预知的风险!


该怎么解决呢，既然问题出在指令有可能重排序上，不让它重排序不就行了，volatile 不就是干这事的吗，我们可以在 instance 变量前面加上一个 volatile 修饰符

```
画外音：volatile 的作用
1.保证的对象内存可见性
2.防止指令重排序
```

优化后的代码如下

```java
public class SingleTonDcl {
    private SingleTonDcl() {
    }

    //在对象前面添加 volatile 关键字即可
    volatile private static SingleTonDcl instance = null;

    public static SingleTonDcl getInstance() {
        if (null == instance) {
            synchronized (SingleTonDcl.class) {
                if (null == instance) {
                    instance = new SingleTonDcl();
                }
            }
        }
        return instance;
    }
}
```

到这里似乎问题已经解决了，双重锁机制 + volatile 实际上确实基本上解决了线程安全问题，保证了“真正”的单例。但真的是这样的吗？继续往下看

## 静态内部类

先看代码

```java
public class SingleTonStaticInnerClass {
    private SingleTonStaticInnerClass() {

    }

    private static class HandlerInstance {
        private static SingleTonStaticInnerClass instance = new SingleTonStaticInnerClass();
    }

    public static SingleTonStaticInnerClass getInstance() {
        return HandlerInstance.instance;
    }
}
```

- 测试代码如下：

```java
class D {
    public static void main(String[] args) {
        IntStream.rangeClosed(1, 5)
                .forEach(i->{
                    new Thread(()->{
                        SingleTonStaticInnerClass instance = SingleTonStaticInnerClass.getInstance();
                        System.out.println("instance = " + instance);
                    }).start();
                });
    }
}

```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de12b557e3544f3db1f4d62b442e224e~tplv-k3u1fbpfcp-zoom-1.image)

静态内部类的特点：

这种写法使用 JVM 类加载机制保证了线程安全问题；由于 SingleTonStaticInnerClass 是私有的，除了 getInstance() 之外没有办法访问它，因此它是懒汉式的；同时读取实例的时候不会进行同步，没有性能缺陷；也不依赖 JDK 版本；

**但是，它依旧不是完美的。**

## 不安全的单例

**上面实现单例都不是完美的**，主要有两个原因

### 1. 反射攻击
首先要提到 java 中让人又爱又恨的反射机制, 闲言少叙，我们直接边上代码边说明，这里就以 DCL 举例（为什么选择 DCL 因为很多人觉得 DCL 写法是最高大上的....这里就开始去”打他们的脸“）

将上面的 DCl 的测试代码修改如下：

```java
class C {
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        Class<SingleTonDcl> singleTonDclClass = SingleTonDcl.class;
        //获取类的构造器
        Constructor<SingleTonDcl> constructor = singleTonDclClass.getDeclaredConstructor();
        //把构造器私有权限放开
        constructor.setAccessible(true);
        //反射创建实例   注意反射创建要放在前面，才会攻击成功，因为如果反射攻击在后面，先使用正常的方式创建实例的话，在构造器中判断是可以防止反射攻击、抛出异常的，
        //因为先使用正常的方式已经创建了实例，会进入if
        SingleTonDcl instance = constructor.newInstance();
        //正常的获取实例方式   正常的方式放在反射创建实例后面，这样当反射创建成功后，单例对象中的引用其实还是空的，反射攻击才能成功
        SingleTonDcl instance1 = SingleTonDcl.getInstance();
        System.out.println("instance1 = " + instance1);
        System.out.println("instance = " + instance);
    }
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2a1959670e8b4a1fa61ff059d7249984~tplv-k3u1fbpfcp-zoom-1.image)

居然是两个对象！内心是不是异常平静？果然和你想的不一样？其他的方式基本类似，都可以通过反射破坏单例。

### 2. 序列化攻击
我们以「饿汉式单例」为例来演示一下序列化和反序列化攻击代码，首先给饿汉式单例对应的类添加实现 Serializable 接口的代码，

```java
public class SingletonHungry implements Serializable {
    private static SingletonHungry instance = new SingletonHungry();

    private SingletonHungry() {
    }

    private static SingletonHungry getInstance() {
        return instance;
    }
}
```
然后看看如何使用序列化和反序列化进行攻击

```java
SingletonHungry instance = SingletonHungry.getInstance();
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("singleton_file")));
// 序列化【写】操作
oos.writeObject(instance);
File file = new File("singleton_file");
ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file))
// 反序列化【读】操作
SingletonHungry newInstance = (SingletonHungry) ois.readObject();
System.out.println(instance);
System.out.println(newInstance);
System.out.println(instance == newInstance);
```
来看下结果
![](https://tva1.sinaimg.cn/large/008eGmZEly1gnok80xoaqj30gc02kaaa.jpg)

果然出现了两个不同的对象！这种反序列化攻击其实解决方式也简单，重写反序列化时要调用的 readObject 方法即可

```java
private Object readResolve(){
    return instance;
}
```
这样在反序列化时候永远只读取 instance 这一个实例，保证了单例的实现。


## 真正安全的单例: 枚举方式

```java
public enum SingleTonEnum {
    /**
     * 实例对象
     */
    INSTANCE;
    public void doSomething() {
        System.out.println("doSomething");
    }
}
```

调用方法

```java
public class Main {
    public static void main(String[] args) {
        SingleTonEnum.INSTANCE.doSomething();
    }
}
```

**枚举模式实现的单例才是真正的单例模式，是完美的实现方式**

有人可能会提出疑问：枚举是不是也能通过反射来破坏其单例实现呢？

试试呗，修改枚举的测试类

```java
class E{
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        Class<SingleTonEnum> singleTonEnumClass = SingleTonEnum.class;
        Constructor<SingleTonEnum> declaredConstructor = singleTonEnumClass.getDeclaredConstructor();
        declaredConstructor.setAccessible(true);
        SingleTonEnum singleTonEnum = declaredConstructor.newInstance();
        SingleTonEnum instance = SingleTonEnum.INSTANCE;
        System.out.println("instance = " + instance);
        System.out.println("singleTonEnum = " + singleTonEnum);
    }
}

```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9edecec6a5324d91bbf2253c0343c9d6~tplv-k3u1fbpfcp-zoom-1.image)

没有无参构造？我们使用 javap 工具来查下字节码看看有啥玄机

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0334aa375d0e4f568574e169a3501ddc~tplv-k3u1fbpfcp-zoom-1.image)

好家伙，发现一个有参构造器 String Int ,那就试试呗

```java
//获取构造器的时候修改成这样子
Constructor<SingleTonEnum> declaredConstructor = singleTonEnumClass.getDeclaredConstructor(String.class,int.class);
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/baf554d9215b481b82e4ee81e785b010~tplv-k3u1fbpfcp-zoom-1.image)

好家伙，抛出了异常，异常信息写着: 「Cannot reflectively create enum objects」

源码之下无秘密，我们来看看 newInstance() 到底做了什么？为啥用反射创建枚举会抛出这么个异常？

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0a920c8d0b748f0a6efa043222787a9~tplv-k3u1fbpfcp-zoom-1.image)

真相大白！如果是枚举，不允许通过反射来创建，这才是使用 enum 创建单例才可以说是真正安全的原因！

## 结束语

以上就是一些关于单例模式的知识点汇总，你还真不要小看这个小小的单例，面试的时候多数候选人写不对这么一个简单的单例，写对的多数也仅止于 DCL，但再问是否有啥不安全，如何用 enum 写出安全的单例时，几乎没有人能答出来！有人说能写出 DCL 就行了，何必这么钻牛角尖？但我想说的是正是这种钻牛角尖的精神能让你逐步积累技术深度，成为专家，对技术有一探究竟的执著，何愁成不了专家?

最后欢迎大家关注我的公号，加我好友:「geekoftaste」,一起交流，共同进步！

![](https://user-gold-cdn.xitu.io/2020/4/29/171c5819f7248204?w=430&h=430&f=jpeg&s=41396)