# GO-Mem-Management

## 1. 存储基础知识
### 1.1存储金字塔

![](https://segmentfault.com/img/remote/1460000020338431)

从上往下依次是：
* CPU寄存器
* Cache
* 内存
* 硬盘等辅助存储设备
* 鼠标等外接设备

存储体系的分层设计：自顶向下，访问速率越来越低，访问时间越来越长，从磁盘到CPU寄存器，上一层可以看做是下一层的缓存。

### 1.2 虚拟内存
虚拟内存分层设计：

![](https://segmentfault.com/img/remote/1460000020338434)

上图展示了进程访问数据，当Cache没有命中的时候，访问虚拟内存获取数据的过程。

访问内存，实际访问的是虚拟内存，虚拟内存通过页表查看，当前要访问的虚拟内存地址，是否已经加在到物理内存，
如果已经在物理内存，则取物理内存数据，如果没有对应的物理内存，则从磁盘加载数据到物理内存，并把物理内存地址和
虚拟内存地址更新到页表。

在没有虚拟内存的时代，物理内存对所有进程是共享的，多进程同时访问同一个物理内存存在并发访问问题。引入虚拟内存后，
每个进程都要各自的虚拟内存，内存的并发访问问题的粒度从多进程级别，可以降低到多线程级别。&oq=在没有虚拟内存的时代，
物理内存对所有进程是共享的，多进程同时访问同一个物理内存存在并发访问问题。引入虚拟内存后，每个进程都要各自的虚拟内存，
内存的并发访问问题的粒度从多进程级别，可以降低到多线程级别。

### 1.3 栈和堆
从虚拟内存，再进一层，看虚拟内存的栈和堆，也就是进程对内存的管理。

![](https://segmentfault.com/img/remote/1460000020338435)

上图展示了一个进程的内存划分，代码中使用的内存地址都是虚拟内存地址，而不是实际的物理内存地址。
栈和堆只是虚拟内存上2块不同功能的内存区域。
* 栈再高地址，从高地址向低地址增长。
* 堆在低地址，从低地址想高地址增长。

**栈和堆相比有这么几个好处：**
1. 栈的内存管理简单，分配比堆上的快。
2. 栈的内存不需要回收，而堆需要。无论是主动free，还是被动的垃圾回收，这都需要花费额外的CPU。
3. 栈上的内存有更好的局部性，堆上内存访问就不那么友好了，CPU访问的2块数据可能在不同的页上，CPU访问
数据的时间可能就上去了。、

### 1.4 堆内存管理
![](https://segmentfault.com/img/remote/1460000020338436)

当我们说内存管理的时候，主要是指堆内存的管理，因为栈的内存管理不需要程序去操心。这小节看下堆内存管理干的是啥，如上图所示主要是3部分：
**分配内存块，回收内存块和组织内存块**。

在一个最简单的内存管理中，堆内存最初会是一个完整的大块，即未分配内存，当来申请的时候，就会从未分配内存，分割出一个小内存块(block)，然后用
链表将所有内存块连接起来。需要一些信息描述每个内存块的基本信息，比如大小(size)，是否使用中(used)和下一个内存块的地址(next)，内存块实际数据存储在data中。

![](https://segmentfault.com/img/remote/1460000020338437)

释放内存实质是把使用的内存块从链表中取出来，然后标记为未使用，当分配内存块的时候，可以从未使用内存块中有先查找大小相近的内存块，如果找不到，
再从未分配的内存中分配内存。

上面这个简单的设计中还没考虑内存碎片的问题，因为随着内存不断的申请和释放，内存上会存在大量的碎片，降低内存的使用率。为了解决内存碎片，
可以将2个连续的未使用的内存块合并，减少碎片。

以上就是内存管理的基本思路，关于基本的内存管理，想了解更多，可以阅读这篇文章《Writing+a+Memory+Allocator》，本节的3张图片也是来自这片文章。

## 2. TCMalloc
TCMalloc是Thread Cache Malloc的简称，是Go内存管理的起源，Go的内存管理是借鉴了TCMalloc，随着Go的迭代，Go的内存管理与TCMalloc不一致地方在不断扩大，
但其主要思想、原理和概念都是和TCMalloc一致的。

在Linux里，其实有不少的内存管理库，比如glibc的ptmalloc，FreeBSD的jemalloc，Google的tcmalloc等等，为何会出现这么多的内存管理库？
本质都是在多线程编程下，追求更高内存管理效率：更快的分配是主要目的。

### 2.1 基本原理

![](https://segmentfault.com/img/remote/1460000020338439)

结合上图，介绍TCMalloc的几个重要概念：

1. **Page**：操作系统对内存管理以页为单位，TCMalloc也是这样，只不过TCMalloc里的Page大小与操作系统里的大小并不一定相等，而是倍数关系。
《TCMalloc解密》里称x64下Page大小是8KB。
2. **Span**：一组连续的Page被称为Span，比如可以有2个页大小的Span，也可以有16页大小的Span，Span比Page高一个层级，
是为了方便管理一定大小的内存区域，Span是TCMalloc中内存管理的基本单位。
3. **ThreadCache**：每个线程各自的Cache，一个Cache包含多个空闲内存块链表，每个链表连接的都是内存块，同一个链表上内存块的大小是相同的，
也可以说按内存块大小，给内存块分了个类，这样可以根据申请的内存大小，快速从合适的链表选择空闲内存块。由于每个线程有自己的ThreadCache，所以ThreadCache访问是无锁的。
4. **CentralCache**：是所有线程共享的缓存，也是保存的空闲内存块链表，链表的数量与ThreadCache中链表数量相同，当ThreadCache内存块不足时，可以从CentralCache取，
当ThreadCache内存块多时，可以放回CentralCache。由于CentralCache是共享的，所以它的访问是要加锁的。
5. **PageHeap**：PageHeap是堆内存的抽象，PageHeap存的也是若干链表，链表保存的是Span，当CentralCache没有内存的时，会从PageHeap取，把1个Span拆成若干内存块，
添加到对应大小的链表中，当CentralCache内存多的时候，会放回PageHeap。如下图，分别是1页Page的Span链表，2页Page的Span链表等，最后是large+span+set，这个是用来保存中大对象的。
毫无疑问，PageHeap也是要加锁的。

![](https://segmentfault.com/img/remote/1460000020338440)

上文提到了小、中、大对象，Go内存管理中也有类似的概念，我们瞄一眼TCMalloc的定义：
1. 小对象大小：0~256KB
2. 中对象大小：257~1MB
3. 大对象大小：> 1MB

小对象分配流程：ThreadCache -> CentralCache -> HeapPage，大部分时候，ThreadCache缓存都是足够的，不需要去访问CentralCache和HeapPage，
无锁分配加无系统调用，分配效率是非常高的。

中对象分配流程：直接在PageHeap中选择适当的大小即可，128 Page的Span所保存的最大内存就是1MB。

大对象分配流程：从large span set中选择合适数量的页面组成span，用来存储数据。

## 3. Go内存管理
前文提到**Go内存管理源自TCMalloc，但它比TCMalloc还多了2件东西：逃逸分析和垃圾回收，这是2项提高生产力的绝佳武器。**

### 3.1 Go内存管理的基本概念
Go内存管理的许多概念在TCMalloc中已经有了，含义是相同的，只是名字有一些变化。

![](https://segmentfault.com/img/remote/1460000020338441)

#### Page
TCMalloc中的Page相同，x64下1个Page的大小是8KB。上图的最下方，1个浅蓝色的长方形代表1个Page。

#### Span
与TCMalloc中的Span相同，**Span是内存管理的基本单位**，代码中为mspan，**一组连续的Page组成1个Span**，
所以上图一组连续的浅蓝色长方形代表的是一组Page组成的1个Span，另外，1个淡紫色长方形为1个Span。

#### mcache
mcache与TCMalloc中的ThreadCache类似，**mcache保存的是各种大小的Span，并按Span+class分类，
小对象直接从mcache分配内存，它起到了缓存的作用，并且可以无锁访问。**

但mcache与ThreadCache也有不同点，TCMalloc中是每个线程1个ThreadCache，Go中是每个P拥有1个mcache，
因为在Go程序中，当前最多有GOMAXPROCS个线程在用户态运行，所以最多需要GOMAXPROCS个mcache就可以保证各线程对mcache的无锁访问，
线程的运行又是与P绑定的，把mcache交给P刚刚好。

#### mcentral
mcentral与TCMalloc中的CentralCache类似，**是所有线程共享的缓存，需要加锁访问，**它按Span+class对Span分类，串联成链表，
当mcache的某个级别Span的内存被分配光时，它会向mcentral申请1个当前级别的Span。

但mcentral与CentralCache也有不同点，CentralCache是每个级别的Span有1个链表，mcache是每个级别的Span有2个链表，
这和mcache申请内存有关，稍后我们再解释。

#### mheap
mheap与TCMalloc中的PageHeap类似，**它是堆内存的抽象，把从OS申请出的内存页组织成Span，并保存起来。**当mcentral的Span
不够用时会向mheap申请，mheap的Span不够用时会向OS申请，向OS的内存申请是按页来的，然后把申请来的内存页生成Span组织起来，
同样也是需要加锁访问的。

但mheap与PageHeap也有不同点：mheap把Span组织成了树结构，而不是链表，并且还是2棵树，然后把Span分配到heapArena进行管理，
它包含地址映射和span是否包含指针等位图，这样做的主要原因是为了更高效的利用内存：分配、回收和再利用。

