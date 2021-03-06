针对扩展性，内存模块做出了很多有意思的尝试。下文中我分成：设计和实现两大部分分别介绍。

# 设计思想

Linux内核的内存管理模块，为了提高自身的扩展性，在设计层面我认为用到了下面几个方法：

* 分类
* 预取
* 专有

虽然可以细分为三种类型，但仔细观察他们的中心思想其实是一样的。用一个字来描述，那就是

> **分**

扩展性之所以会出现瓶颈是因为生产者和消费者之间的不匹配。假如能够保证生产者和消费者数量是1:1，那么就不会产生扩展性问题。然而现实中往往出现这样的不匹配。基于这个思维出发，那么能想到的一个纬度就是尽可能增加生产者来减少消费者之间的竞争。

接下来我们来看看内存管理模块是如何在这三个方面增加生产者的。

## 分类

假设我们把整个物理内存空间看成一个整体（生产者），那么面对当前系统中数以千记的进程（消费者）这个扩展性一定是很差的。那么第一个增加生产者的方式就是给内存分类。

在当前内存设计中，我能想到的分类依据有：

* NUMA
* ZONE
* 页大小

如果舍前两者是从内存的物理属性出发来分类，那么第三种则是软件设计人员创造性的增加了一种分类的方式。

内存按照NUMA,ZONE进行分类的示意图如下：

```
   node_data[0]                                                node_data[1]
   +-----------------------------+                             +-----------------------------+        
   |node_id                <---+ |                             |node_id                <---+ |        
   |   (int)                   | |                             |   (int)                   | |        
   +-----------------------------+                             +-----------------------------+    
   |node_zones[MAX_NR_ZONES]   | |    [ZONE_DMA]               |node_zones[MAX_NR_ZONES]   | |    [ZONE_DMA]       
   |   (struct zone)           | |    +---------------+        |   (struct zone)           | |    +---------------+
   |   +-------------------------+    |0              |        |   +-------------------------+    |empty          |
   |   |                       | |    |16M            |        |   |                       | |    |               |
   |   |zone_pgdat         ----+ |    +---------------+        |   |zone_pgdat         ----+ |    +---------------+
   |   |                         |                             |   |                         |        
   |   |                         |    [ZONE_DMA32]             |   |                         |    [ZONE_DMA32]        
   |   |                         |    +---------------+        |   |                         |    +---------------+   
   |   |                         |    |16M            |        |   |                         |    |3G             |   
   |   |                         |    |3G             |        |   |                         |    |4G             |   
   |   |                         |    +---------------+        |   |                         |    +---------------+   
   |   |                         |                             |   |                         |        
   |   |                         |    [ZONE_NORMAL]            |   |                         |    [ZONE_NORMAL]       
   |   |                         |    +---------------+        |   |                         |    +---------------+   
   |   |                         |    |empty          |        |   |                         |    |4G             |   
   |   |                         |    |               |        |   |                         |    |6G             |   
   +---+-------------------------+    +---------------+        +---+-------------------------+    +---------------+
```

可以看到，内存管理模块首先将内存按照numa分成了两个node。这样当有消费者需要分配/释放内存时，只需要找到相应的node。如果两个消费者需要释放存在不同node上的内存，那他们就可以同时进行。

进一步的又将每个node上的内存按照zone的属性进行了划分，原理同上。

关于node，zone的概念大家可以在[Node-Zone-Page][1]一文中找到更多的信息。

内存按照页大小进行分类的示意图如下：

```
      struct zone
      +------------------------------+      The buddy system
      |free_area[MAX_ORDER]  0...10  |
      |   (struct free_area)         |
      |   +--------------------------+
      |   |nr_free                   |  number of available pages
      |   |(unsigned long)           |  in this zone
      |   |                          |
      |   +--------------------------+
      |   |                          |           free_area[0]
      |   |free_list[MIGRATE_TYPES]  |  Order0   +-----------------------+
      |   |(struct list_head)        |  Pages    |free_list              |
      |   |                          |           |  (struct list_head)   |
      |   |                          |           +-----------------------+
      |   |                          |
      |   |                          |           free_area[1]
      |   |                          |  Order1   +-----------------------+
      |   |                          |  Pages    |free_list              |
      |   |                          |           |  (struct list_head)   |
      |   |                          |           +-----------------------+
      |   |                          |
      |   |                          |              .
      |   |                          |              .
      |   |                          |              .
      |   |                          |
      |   |                          |
      |   |                          |           free_area[10]
      |   |                          |  Order10  +-----------------------+
      |   |                          |  Pages    |free_list              |
      |   |                          |           |  (struct list_head)   |
      |   |                          |           +-----------------------+
      |   |                          |
      +---+--------------------------+
```

这个示意图将zone结构体进一步打开。其中重要的就是free_list成员。这是一个按照页大小排列的数组，不同大小的页会保存在对应大小的数组中。这样以来当需要分配内存时，我们只需要在对应的大小数组中取搜索。以此提高了扩展性。

关于free_list的概念大家也可以在[Node-Zone-Page][1]一文中找到更多的信息。

## 预取

这第二种方式是预取，直接的意思是每次要获取资源时，并不是获取所需的数量，而是获取多一些的数量来减少以后获取资源的次数。从另一个角度理解，就好比将一部分的资源单独分出来专门给某个消费者使用。

在内存管理模块总，这个方式是建立在第一种分类基础之上的。

比如有一个消费者每次都申请一定大小的内存，如果每次申请时都从free_list上分配，那么每次都会和其他需要申请内存的消费者发生竞争。但是如果我们每次申请时多申请一部分，那么竞争的次数就会大大下降。从而提高了系统的扩展性。

在内存管理中使用到这个思想的有：

* per_cpu_pageset
* slub

对于per_cpu_pageset，可以观察这个结构体中的batch字段。

```
    +------------------------------+
    |pageset                       |
    |   (struct per_cpu_pageset *) |
    |   +--------------------------+
    |   |pcp                       |
    |   |  (struct per_cpu_pages)  |
    |   |  +-----------------------+
    |   |  |count                  |
    |   |  |high                   |
    |   |  |batch                  |
    |   |  |                       |
    |   |  |lists[MIGRATE_PCPTYPES]|
    +---+--+-----------------------+
```

这个字段控制了每次需要从页分配器中获取/释放的页面个数，具体可以参考下面这两个函数。

  * __rmqueue_pcplist()。
  * free_unref_page_commit(）。

当per_cpu_pageset空时，系统是从free_list上取下batch个页面，而不是仅仅取出一个。当per_cpu_pageset满时，则是反向操作。这样就减少了对free_list的竞争。

对于slub，可以观察结构体中的oo字段。

```
    +------------------------------+
    |oo                            |
    |min                           |
    |max                           |
    |   (kmem_cache_order_objects) |
    |   +--------------------------+
    |   |order                     |
    |   |order_objects             |
    +---+--------------------------+
```

这个oo表示了每次分配一个slab时，需要的page的页数。其中计算个过程和使用分别在下面两个函数中。

  * calculate_sizes()
  * alloc_slab_page()

有兴趣的朋友可以进一步学习。

## 专有

归根结底扩展性的问题就是**生产者和消费者之间的不匹配**，无论是哪一方速度跟不上就造成了性能上的损失，或者从另一个角度上看就是某种资源的闲置。

举一个不太恰当的例子：

> 小明家里有一台电视机，小明的爸爸喜欢看新闻，小明的妈妈喜欢看电视剧，小明喜欢看动画片。
> 每次要看电视的时候，一家三口就开始上演遥控器争夺战。当然小明是直接出局的，因为小明被要求不能看动画片，必须写作业。。。
> 通常这个问题的解决方法很简单，**再买一个电视机**。

故事讲完了，回到内存模块的设计，这个原理同样可以适用。

> 在必要的时候，给每个关键的资源分配专属的资源，用来减少竞争，充分利用资源，提高扩展性。

在内存管理中使用到这个思想还是这两个：

* per_cpu_pageset
* slub

不过使用的具体方式上略有不同。

对于per_cpu_pageset，我们可以看一下这个示意图：

```
    struct zone
    +------------------------------------------------------------------------------------------------+
    |pageset                                                                                         |
    |   (struct per_cpu_pageset *)                                                                   |
    |   cpu0                          cpu1                                cpuN                       |
    |   +--------------------------+  +--------------------------+  ...   +--------------------------+
    |   |pcp                       |  |pcp                       |        |pcp                       |
    |   |  (struct per_cpu_pages)  |  |  (struct per_cpu_pages)  |        |  (struct per_cpu_pages)  |
    |   |  +-----------------------+  |  +-----------------------+        |  +-----------------------+
    |   |  |count                  |  |  |count                  |        |  |count                  |
    |   |  |high                   |  |  |high                   |        |  |high                   |
    |   |  |batch                  |  |  |batch                  |        |  |batch                  |
    |   |  |                       |  |  |                       |        |  |                       |
    |   |  |lists[MIGRATE_PCPTYPES]|  |  |lists[MIGRATE_PCPTYPES]|        |  |lists[MIGRATE_PCPTYPES]|
    +---+--+-----------------------+--+--+-----------------------+--------+--+-----------------------+
```

之前我们说过，在分配页时，会找到对应的zone上对应的freelist。但这么做就会产生竞争。内核为了进一步减少竞争为每个cpu建立一个order 0的页集合(pageset)。这样当需要分配一个页时，每个cpu上都有自己的资源不需要和别人抢。只有当自己资源没有的时候，才会去zone上找。用这个方法减少了竞争。

对于slub，我们可以看下面这个示意图。

```
            kmem_cache                      
            +------------------------------+
            |name   (char *)               |
            +------------------------------+
            |cpu_slab                      |    __percpu
            |   (struct kmem_cache_cpu*)   |
            |   +--------------------------+
            |   |tid                       |
            |   |          (unsigned long) |
            |   |freelist  (void **)       |
            |   |page      (struct page *) |
            |   |partial   (struct page *) |
            +---+--------------------------+
            |node[MAX_NUMNODES]            |
            |   (struct kmem_cache_node*)  |
            |   +--------------------------+
            |   |list_lock                 |
            |   |    (spinlock_t)          |
            |   |nr_partial                |
            |   |    (unsigned long)       |
            |   |partial                   |
            |   |    (struct list_head)    |
            +---+--------------------------+
```

在这张图上值得注意的是两个地方

  * cpu_slab
  * node

前者是一个percpu变量，后者是给每个numa node都分配的链表。

原理上和前面的一样，这里我就不做赘述了。

# 实现细节

第一部分讲了三种扩展性的应对方式，接下来讲讲我观察到的三种实现临界资源保护的方法。

这三种方式没有绝对的好坏，在不同的场景下选择相应的方式。

## 关中断

关中断是一个非常霸道的方式，通常情况下是不建议这么做的，但是有些情况除外。

比如在slub分配器中，分配过程中的某一段就采用了这种方式。代码片段如下：

```
    __slab_alloc()
      local_irq_save()
      ___slab_alloc(s->cpu_slab)
      local_irq_restore()
```

之所以这段代码采用关中断的方式来保护，是因为___slab_alloc()中将要处理cpu_slab这个per cpu变量。

因为我们知道其他cpu是不会访问到这个资源的，而我们只需要保证当前cpu不会被其他进程/中断打断即可起到保护临界资源的作用。这样就不必使用以下两种方式再来保护cpu_slab中的资源了。

## 锁

锁应该是大家最为常见的保护方式了，这里我就不再赘述。只列举几个在内存管理模块中常见的锁。

  * zone->lock
  * kmem_cache_node->list_lock

## cmpxchg

这个就是传说中无锁化的操作了。

比如这个改动

```
commit 8a5ec0ba42c4919e2d8f4c3138cc8b987fdb0b79
Lockless (and preemptless) fastpaths for slub
```

具体的内容就在这个补丁中了，据说效果优化了40%。

这个神操作还有待继续研究～

[1]: /mm/05-Node_Zone_Page.md
