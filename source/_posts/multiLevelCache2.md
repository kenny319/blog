---
title: 多级缓存 （下）- 堆外缓存
---
## 写在开头
>上文介绍了多级缓存的背景以及多级缓存带来的一些新的问题，本文将重点介绍本地缓存。

>通过本文您可以了解到：


>* 本地缓存的一些开源解决方案
>* 堆外缓存的一种实现OHC
>* 使用堆外缓存需要注意的问题：序列化、内存管理、容量评估等

## 开源实现

本地缓存的开源实现也有不少，对于堆内缓存的开源方案则更多，比如Guava或者Ehcache，这个选择相对比较容易。这里我们重点关注的是堆外缓存的开源实现。了解到的主要有：

* Ehcache 3.0：3.0基于其商业公司一个非开源的堆外组件的实现
* Chronical Map：LGPL V3的license, 高级特性非开源
* OHC：来源于Cassandra 3.0， Apache v2
* Ignite: 一个规模宏大的内存计算框架，堆外缓存只是它的冰山一角的一角，Apache项目

选型主要考虑成熟度、License、功能、性能、代码质量等等，这里不做详细的方案比较。如果需要进行定制化，一个满足需求的简单且能够驾驭的框架可能是更好的选择。

本文后续的内容以OHC作为基础进行介绍。


## 堆外缓存关注点

### 序列化
从上一节我们大概可以了解到，存放一个JAVA对象到堆外缓存中需要有一个从JAVA对象到ByteBuffer的转换的过程。而为了尽量简化API的使用，使用者不需要去为对象编写特定的序列化过程，我们需要一个高效的序列化框架。先通过序列化框架序列化, 再转成堆外的ByteBuffer上。在堆外缓存的使用场景中，该序列化框架应该至少具备以下一些要求：

>#### 高性能
>这个不用说
>#### 序列化的数据overhead小
>由于堆外缓存本来就占用内存资源多，所以如果序列化框架序列化造成的overhead过大的话是无法接受的。其实如果序列化框架造成的overhead是定长的话是最完美的，少去了中间转换的过程，可以直接把JAVA对象序列化到堆外，大大提高了性能也能降低GC的频率。
>#### 简单易用
>使用要尽量简单，尽量少的倾入，最好是用户根本就不知道你用的是什么鬼。所以那些基于IDL的如PB, Thrift，以及基于Schema的如Avro就不是很好的选择，如果还要代码生成的就要崩溃了, 这些大部分是基于跨语言通讯的需要。
>#### 兼容性
>这里的兼容性包括几个方面：

>* 序列化协议的兼容性：不同版本间序列化的协议尽量要稳定，不然就放的进去读不出来了。这里还要考虑向前向后兼容，至少要向后兼容即老版本写的数据新版本可以读的出来。
>* 数据模型的兼容性：如果你的数据模型发生了变化，比如：增加属性、减少属性、更改类型等等，随着业务的发展这是非常平常的事。如果序列化框架默认无法直接支持，那最好有方法让用户自己做兼容性标记，如读的时候可以忽略增加或者减少的属性等等。
>* 当然如果你的数据没有持久化又或者你的应用不支持热加载如OSGI可以不要考虑该问题。

关于序列化协议的选择，有一个非常不错的benchmark可以参考 [jvm-serializers](https://github.com/eishay/jvm-serializers/wiki)，里面对各种序列化协议从各种不同维度进行了对比。最后我们选择了Kryo，因为Kryo在性能上以及序列化造成的overhead上均表现非常优异，且使用对用户透明。

一个Kryo的使用例子：

``` java
// Setup ThreadLocal of Kryo instances
private static final ThreadLocal<Kryo> kryos = new ThreadLocal<Kryo>() {
    protected Kryo initialValue() {
        Kryo kryo = new Kryo();
        // configure kryo instance, customize settings
        return kryo;
    };
};

Kryo kryo = kryos.get();
    // ...
Output output = new Output(new FileOutputStream("file.bin"));
SomeClass someObject = ...
kryo.writeObject(output, someObject);
output.close();
// ...
Input input = new Input(new FileInputStream("file.bin"));
SomeClass someObject = kryo.readObject(input, SomeClass.class);
input.close();
```

这里自己需要简单封装一下，因为Kryo不是线程安全的，且每次使用创建的成本较高，所以要么Pooling要么TLC.

### 内存管理
#### JAVA堆外内存分配

堆外内存的分配大家比较熟悉的是使用如下的JNI方式进行分配：

``` java
java.nio.ByteBuffer.allocateDirect(int)           
```

大抵的原理是其内部还是使用的Unsafe进行的内存分配, 从下面的API声明可知该方法是一个JNI的封装。详细请参考：[Unsafe](http://www.docjar.com/html/api/sun/misc/Unsafe.java.html)。

~~~ java
public native long allocateMemory(long bytes)
~~~

通过Unsafe进行分配的内存受限于XX:MaxDirectMemorySize的配置，换句话无论哪里用到了堆外缓存，只要通过Unsafe的方式进行的分配都是共享该Quota的。最好的方式其实是不同用途的堆外内存可以隔离开来，比如堆外缓存的一块，netty的一块。

JNA（Java Native Access) 是社区开发的一套类库，号称是JNI的终结者。传统需要调用本地方法既要写c代码生成DLL/SO，又要写java代码进行封装才能提供JAVA调用，既繁琐效率又低。而JNA是通过libffi来实现的，提供了一套接口，用户只需要通过JAVA代码定义本地方法和数据类型，JNA负责把JAVA方法的调用进行Dispatch到相应的本地方法调用，你不再需要不厌其烦的为每个本地方法都写个C和JAVA的wrapper方法对。限于篇幅，请参考另外一篇。

先来看一下，通过JNA的malloc是怎么做的：

~~~java
com.sun.jna.Native.malloc(long)
~~~

方法声明如下：

~~~java
public static native long malloc(long size);
~~~

通过JNA不仅开发效率更高，性能上也有了提高，以malloc为例。在我本地的mac(2.7 GHz Intel Core i5)上做了一个malloc和free的组合测试，JNA的性能是Unsafe的两倍以上。

由于使用JNA进行堆外内存的分配，完全绕过了Bits的内存大小的管理即不会占用XX:MaxDirectMemorySize的空间，所以只要物理内存上足够，你完全不需要为了引入堆外缓存而更改原有的XX:MaxDirectMemorySize配置，同时也起到了较好的隔离。

这里还有一个关于[JNA vs Unsafe](http://mail.openjdk.java.net/pipermail/hotspot-dev/2015-February/017089.html)的讨论, 有兴趣的也可以看一下。

#### 本地内存分配管理
除了要有一个高效的JAVA堆外分配的方法之外，一个高效的本地内存分配管理的策略和库也非常重要。OHC就强烈推荐使用Jemalloc替换glibc的malloc.

网上有各种相关的性能评测报告和原理分析，Jemalloc在现代多核的处理器架构的情况下性能表现较为优异，主要得益于它的Thread Local Cache的引入，在类似JAVA的TLAB(Thread local allocation buffer)的内存管理策略，在分配小内存的时候可以直接在TLC里分配，减少锁的竞争。但是据说在存在大量的小内存分配的时候，额外的内存浪费会比google的tcmalloc大，具体的没有详细研究，以后可以找时间详细对比测试一下。可以参考：
[ptmalloc vs tcmalloc vs jemalloc](http://www.360doc.com/content/13/0915/09/8363527_314549128.shtml), 以及tcmalloc和jemalloc的理论基础[A Scalable Concurrent malloc(3) Implementation for FreeBSD](https://people.freebsd.org/~jasone/jemalloc/bsdcan2006/jemalloc.pdf)。


### OHC内部结构

OHC默认的实现其实就是一个堆外的ConcurrentHashMap,大致结构如下：

![OHC Data structure](/images/multilevelcache2/OHC-data.jpg)

Segment的数量默认是内核的两倍，每个Segment的bucket数量默认是8192，loadFactor是0.75，应用需要根据自己的key的数量来合理的设置以上参数，否则可能会导致rehash次数过多或者访问效率过低。实现方式跟普通的ConcurrentHashMap差不多，只是从上图可知每个bucket及每个entry的数据都存放在一个特定的堆外数据结构当中。每个bucket 8个字节存放这个bucket的首个entry的内存地址，每个entry也是一块连续的内存，由固定长度的64个字节的头和key length+value length的data组成。

当往cache中放一个k/v时，会动态申请一块新的内存，删除后直接释放。为什么没有像netty一样通过维护一组不同大小的buffer来组合复用内存呢？个人觉得cache主要解决的是读多写少的问题，数据变的频率较低。另外堆外缓存存放的是百万甚至千万的entry，所以对内存大小比较敏感，最好是一点也不浪费，要多少则申请多少，而不是使用固定大小的内存块，大小总归不一定匹配，会有一些浪费。

最后，OHC也提供了一个参数用于配置堆外缓存的总内存大小，如果你是需要缓存全量数据的则使用的时候需要合理评估容量进行设置。如果缓存的是热点数据，当空间不足时会根据LRU进行淘汰数据。

### 容量评估

当你使用堆外缓存时，一个很重要的就是需要提前进行内存容量的评估。这里有个需要重点考虑的因数是除了业务数据本身的大小之外，你还要考虑序列化框架的序列化和缓存框架本身的数据结构造成的overhead。比如：kryo需要可能需要存储对象的类名、属性类型、属性是否为空等各种标识，取决于你的对象的复杂程度，会造成额外的overhead. 另外，以OHC为例，由OHC的数据结构可知，OHC造成的overhead = bucket number \* 8字节 + entry number \* 64个字节。

当然最稳妥的方式是根据业务情况进行压测！

## 写在最后

后续还可以关注以下问题：

* 评测缓存数据量及更新频率对GC的影响，为选择堆内还是堆外缓存提供参考依据。
* JNA + Jemalloc vs Unsafe vs ptmalloc 的性能评测。
* 如何从框架层面简化多级缓存的使用

就这么多吧！