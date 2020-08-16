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

#### 大小转换
![](https://segmentfault.com/img/remote/1460000020338442)

*object size*: 代码里简称size，指申请内存的大小

*size class*： 代码里简称class，它是size的级别，相当于把size归类到一定大小的区间段，比如size[1,8]属于size class 1，size(8,16]属于size class 2。

*span class*: 指span的级别，但span class的大小与span的大小并没有正比关系。span class主要用来和size class做对应，1个size class对应2个span class，2个span class的大小相同，只是功能不同，1个用来存放包含指针的对象，一个用来存访不包含指针的对象，不包含指针对象的span就无需GC扫描了。

*num of page*：代码里进程npage，代表page的数量，其实就是span包含的页数，用来分配内存。

### 3.1 Go内存分配
Go中的内存分类并不像TCMalloc那样分成小、中、大对象，但是它的小对象里又细分了一个Tiny对象，Tiny对象指大小在1Byte到16Byte之间并且不包含指针的对象。小对象和大对象只用大小划定，无其他区分。

![](https://segmentfault.com/img/remote/1460000020338446)

小对象是在mcache中分配的，而大对象是直接从mheap分配的，从小对象的内存分配看起。

#### 小对象分配
![](https://segmentfault.com/img/remote/1460000020338441)

大小转换这一小节，我们介绍了转换表，size+class从1到66共66个，代码中_NumSizeClasses=67代表了实际使用的size class数量，即67个，从0到67，size class 0实际并未使用到。上文提到1个size class对应2个span class。numSpanClasses为span+class的数量为134个，所以span class的下标是从0到133，所以上图中mcache标注了的span class是，span class 0到span class 133。每1个span class都指向1个span，也就是mcache最多有134个span。

##### 为对象寻找span
1. 计算对象所需内存大小size
2. 根据size到size class的映射，计算所需的size class
3. 根据size class和对象是否包含指针算出span class
4. 获取该span class指向的span。

##### 从span分配对象空间
Span可以按对象大小切成很多份，这些都可以从映射表上计算出来，以size class 3对应的span为例，span大小是8KB，每个对象实际所占空间为32Byte，这个span就被分成了256块，可以根据span的起始地址计算出每个对象块的内存地址。

![](https://segmentfault.com/img/remote/1460000020338447)

随着内存的分配，span中的对象内存块，有些被占用，有些未被占用，比如上图，整体代表1个span，蓝色块代表已被占用内存，绿色块代表未被占用内存。

当分配内存时，只要快速找到第一个可用的绿色块，并计算出内存地址即可，如果需要还可以对内存块数据清零。

##### span没有空间怎么分配对象
span内的所有内存块都被占用时，没有剩余空间继续分配对象，mcache会向mcentral申请1个span，mcache拿到span后继续分配对象。

##### mcentral向mcache提供span
mcentral和mcache一样，都是0~133这134个span class级别，但每个级别都保存了2个span list，即2个span链表：
1. nonempty：这个链表里的span，所有span都至少有1个空闲的对象空间。这些span是mcache释放span时加入到该链表的。
2. empty：这个链表里的span，所有的span都不确定里面是否有空闲的对象空间。当一个span交给mcache的时候，就会加入到empty链表。

![](https://segmentfault.com/img/remote/1460000020338448)

实际代码中每1个span+class对应1个mcentral，图里把所有mcentral抽象成1个整体了。

mcache向mcentral要span时，mcentral会先从nonempty搜索满足条件的span，如果每找到再从emtpy搜索满足条件的span，然后把找到的span交给mcache。

##### mheap的span管理
mheap里保存了2棵二叉排序树，按span的page数量进行排序：
1. free：free中保存的span是空闲的并且非垃圾回收的span。
2. scav：scav中保存的是空闲并且已经垃圾回收的span。

如果是垃圾回收导致的span释放，span会被加入到scav，否则加入到free，比如刚从OS申请的的内存也组成的Span。

![](https://segmentfault.com/img/remote/1460000020338449)

mheap中还有arenas，有一组heapArena组成，每一个heapArena都包含了连续的pagesPerArena个span，这个主要是为mheap管理span和垃圾回收服务。

mheap本身是一个全局变量，它其中的数据，也都是从OS直接申请来的内存，并不在mheap所管理的那部分内存内。

##### mcentral向mheap要span
mcentral向mcache提供span时，如果emtpy里也没有符合条件的span，mcentral会向mheap申请span。

mcentral需要向mheap提供需要的内存页数和span class级别，然后它优先从free中搜索可用的span，如果没有找到，会从scav中搜索可用的span，如果还没有找到，它会向OS申请内存，再重新搜索2棵树，必然能找到span。如果找到的span比需求的span大，则把span进行分割成2个span，其中1个刚好是需求大小，把剩下的span再加入到free中去，然后设置需求span的基本信息，然后交给mcentral。

##### mheap想OS申请内存
当mheap没有足够的内存时，mheap会向OS申请内存，把申请的内存页保存到span，然后把span插入到free树。

在32位系统上，mheap还会预留一部分空间，当mheap没有空间时，先从预留空间申请，如果预留空间内存也没有了，才向OS申请。

#### 大对象分配
大对象的分配比小对象省事多了，99%的流程与mcentral向mheap申请内存的相同，所以不重复介绍了，不同的一点在于mheap会记录一点大对象的统计信息，见mheap.alloc_m()。