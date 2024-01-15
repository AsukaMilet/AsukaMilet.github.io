+++
title = "Are You Sure You Want to Use MMAP in Your Database"
date = 2023-11-14
draft = false
author = "AsukaMilet"
tags = ["Database", "Paper"]
categories = "Paper"
description = "Andy他们花费了大量努力让你相信在DBMS中使用mmap是一个坏主意，为什么要继续坚持呢"

+++

Memory-mapped(mmap) I/O允许OS直接将位于non-volatile设备上的文件内容映射至用户的地址空间，进程可以通过指针来访问其文件，就好像文件本身驻留在内存中一样，而在幕后，OS默默地将page加载至内存，并在给定的mmap buffer已满时，进行换页(demand paging)。

由于mmap的上述特性，为DBMS开发人员提供了一个传统的buffer pool实现的替代选项，因此相当一部分的DBMS使用mmap来支持disk-based DBMS，然而mmap存在它潜在的风险。Andy他们发表在CIDR的这篇兼备娱乐与科普性质的文章就是为了告诫DBMS开发者不要图一时之快，mmap并不是传统buffer pool的完美替代品。

总之，一句话来总结：MMAP = 💩

----

## Background

对于disk-based DBMS，传统的buffer pool提供了数据库完全驻留在物理内存中的假象，DBMS通过`read`/`write`这样的system calls来与I/O设备(主要是HDD/SSD等non-volatile存储设备。为简短起见，后面统称硬盘)进行交互。通过文件I/O机制，数据在user space中的buffer与I/O设备之间进行来回的拷贝，DBMS拥有对何时以及如何进行page转移的完整控制权。

作为替代，DBMS也可以将数据的移动控制权给予OS，OS来负责文件映射以及它自身的page cache。POSIX `mmap` system call就提供了这一实现。对于`mmap`的详述，由于这不是关于OS的文章，恰恰相反，这是一篇讲OS坏话的文章，因此建议寻找一本OS相关的书籍来详细了解。

从表面上看，`mmap`为DBMS管理文件I/O提供了一个充满吸引力的选项，通过`mmap`:

1. 易于使用并且降低了开发成本, DBMS可以直接通过指针访问驻留在硬盘上的数据，就好像它们直接驻留在内存一样
2. 从性能角度，`mmap`减少了syscall的开销(i.e., `read`/`write`)

由于这些"显而易见"的好处，许多DBMS都选择了使用`mmap`来作为传统buffer pool的实现，然而实际上`mmap`存在巨大的缺陷，既包括数据安全性的缺陷，也包括性能上的缺陷，原始论文的目的就是告诫DBMS kernel开发人员`mmap`并不如想象中的那般美好。

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/mmap_based_DBMSs.png" style="zoom:50%;margin: 0 auto" />

## 从拥抱MMAP到走向毁灭

### MMAP基本原理

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/how_using_mmap.png" style="zoom: 50%;" >

为了明白为什么`mmap`可以作为buffer pool的替代选项，以及为什么`mmap`会让生活变得不幸，我们首先需要对`mmap`的基本工作机制有一个认知，以上图为例：

1. DBMS调用`mmap`并获取一个指向对应文件内容的指针
2. OS在进程的虚拟地址空间中为该memory-mapped file分配空间，但并不将实际数据加载至物理内存中(demand paging)
3. DBMS通过指针尝试访问文件
4. OS检索TLB和page table
5. 由于物理内存中没有与virtual address space对应的映射，因此OS触发缺页异常，将DBMS所引用到的page加载至物理内存中
6. OS设置page table
7. TLB缓存对应的address translation

通过上述的基本流程我们可以发现，OS维护它自己的page cache，当page cache已满，又有新页需要载入物理内存时，OS会自动地为我们选择不需要的页进行替换，并将对应的地址转换映射从页表和TLB中进行清除。不仅如此，OS还要保证不同processors的TLB也会得到相应的刷新，这意味潜在的处理器之间的同步开销

> 关于TLB flush时的处理器之间的同步开销，可以参见[以前的文章](../../articles/what-every-programmer-should-know-about-memory-cache篇/)

在介绍完`mmap`的工作原理后，POSIX为`mmap`提供了一组基本的API，参见[mmap-Linux manual page](https://man7.org/linux/man-pages/man2/mmap.2.html)，应重点关注`madvise`, `mlock`, `msync`这几个API

### 走向毁灭

如上面的图表所展示的，许多DBMS都选择了`mmap`作为buffer pool的替代。但随着时间的推移，越来越多的DBMS发现`mmap`带来了不幸，最终还是抛弃了`mmap`，在这里仅简单列举部分抛弃了`mmap`的DBMS

1. MongoDB
2. [InfluxDB](https://www.influxdata.com/blog/announcing-influxdb-iox/)
3. [SingleStore](https://www.singlestore.com/blog/linux-off-cpu-investigation/)
4. [RocksDB](https://rocksdb.org/docs/support/faq.html)

----

## 隐藏在MMAP背后的问题

在介绍完基础的背景知识之后，现在我们可以详细阐述为什么`mmap`并不是buffer pool的一个优良替代选项

### Transactional Safety

`mmap`最大的问题就是OS能够随时将某个脏页flush至硬盘而不管该txn是否commit，当flush发生时，DBMS无能为力。因此mmap-based DBMS必须实现复杂的机制确保OS的flush不会破坏transactional safety，原文将不同的实现主要分为了三大类：

1. OS Copy-On-Write(貌似只有MongoDB MMAPv1)

   DBMS在virtual memory space中通过`mmap`创建同一个文件的两个copies，指向同一个physical memory page。第一个作为主要的copy, 第二个作为private space用于暂存txn的写操作。DBMS通过`MAP_PRIVATE` flag来创建private space

   如何工作：DBMS将txn牵涉到的page在private space中进行修改，OS隐式地将修改的内容拷贝至新的physical memory，并将page table指向新的physical memory。primary copy不会看到这些修改，因此OS也不会将其同步至硬盘。为了持久化，当WAL同步至硬盘后使用一个daemon thread来将修改应用于primary copy

   缺陷：
   - 存在冲突的txns时(e.g 两个txn需要修改同一个页面)，DBMS必须确保commited txn的修改同步至primary copy，意味着需要额外追踪不同的txns待更新的页面
   - 随着修改的增多，private workspace可能会逐渐增长，此时virtual memory address space中可能存在两份database的copy

2. User-Space Copy-On-Write(SQLite, MonetDB, RavenDB)

   DBMS手动的将**被修改的页面**(建议仔细琢磨，与上面的OS COW进行对比)拷贝至一个自身维护的buffer(in virtual memory address space)

   如何工作：DBMS只需要对WAL执行写操作，并对buffer中的拷贝执行修改。在txn commit, WAL持久化之后可以安全地将修改应用至`mmap`维护的区域，由于将entire pages拷贝至对应区域开销太大，某些DBMS支持直接将WAL应用于`mmap`维护的区域

3. Shadow paging(LMDB)

   ~~这个原理就不必多说了吧~~

   缺陷：LMDB只允许一个写者(txn级别)，等价于锁DB

### I/O阻塞

传统的buffer pool支持异步I/O避免query执行时阻塞threads。例如：B+ tree leaf nodes的顺序扫描允许异步的发起read requests来减小延迟，但`mmap`不支持异步读写，如果`mmap`访问未命中时，当前thread会因为触发缺页异常而被阻塞。

不仅如此，由于DBMS不知道哪些页面位于物理内存中，访问任意页面都有可能触发意料之外的I/O阻塞。

对于该问题，DBMS开发人员可以使用OS提供的接口来减轻这一问题：

1. 使用`mlock`来pin住页面 

   缺陷：OS对于单个进程能够pin住的页面数量有限制。并且DBMS还要仔细地追踪并unpin页面使OS能够剔除不需要的页面

2. 使用`madvise`来提示OS查询可能的访问模式，例如使用`MADV_SEQUENTIAL`来告诉OS要执行sequential scan

   缺陷：OS可能不听你的指示(~~真是一个悲伤的故事~~)，并且一旦提示出现错误，例如实际存在大量random I/O结果提示了sequential I/O那就完蛋了

3. 创建额外的线程来prefetch

尽管这些方法减缓/部分减缓上述问题，但仍另外引入了显著的复杂性，这与使用`mmap`的初衷相违背(别忘了，你使用`mmap`就是为了避免实施buffer pool带来的痛苦)

### 错误处理

DBMS的一个核心职责就是保障数据的完整性，因此当数据出现错误时进行处理至关重要。当DBMS使用`mmap`时，可能存在以下问题:

1. 一些DBMSs在page-level维护一份checksum来检查I/O期间数据是否被污染。当从硬盘中读取某个page时，DBMS会使用存储在page metadata中的checksum来验证page中的内容。如果使用`mmap`，DBMS每次访问某个page时都需要验证其校验和，因为自上次访问以来，OS可能默默地将该page剔除出物理内存，导致发生了I/O(如果使用buffer pool, DBMS就能手动控制哪些页面维持在物理内存中，不会被剔除)
2. 许多DBMSs都是用内存不安全的语言实现的(~~说的就是C/C++~~), 意味着指针出现错误时可能会污染内存中pages的内容，手动实现buffer pool时DBMS会在将这些pages写回硬盘前进行检查，但使用`mmap`时OS可能会默默地将受到污染的数据写回硬盘
3. 基于`mmap`的数据库实现中，任何与`mmap`进行交互的模块都有可能产生`SIGBUS`信号，DBMS开发人员只能通过复杂的signal handler来进行处理

### 性能问题

最后，`mmap`带来的最大也是最显著的问题就是性能瓶颈，尽管Andy他们认为上述如错误处理等问题可以通过DBMS开发小心翼翼的实现来克服，但是`mmap`带来的性能瓶颈在不修改OS kernel的情况下无法避免。

#### 理想情况

理论上使用`mmap`代替buffer pool会有更优越的性能，因为`mmap`主要避免了两方面的开销：
1. `mmap`规避了显式调用`read/write` syscall的开销，因为通过一次`mmap` syscall，OS在背后默默地处理缺页和文件映射
2. `mmap`可以返回存储在OS page cache中的指针，避免了额外的拷贝至user space buffer的开销。作为附加的好处，还能减少内存的花费，因为data无需在user space中重复存在

#### 实际情况

由于理论上使用`mmap`的好处这么大，那么随着SSD的发展(e.g PCIe 5.0 NVMe)，使用`mmap`与传统buffer pool之间的性能差距应该越来越大，但实际上OS的page evict机制在高带宽的硬盘设备上针对超过大于内存的DBMS工作负载时，无法很好的扩展到多个线程 (原文: cannot scale beyond a few threads for larger-than-memory DBMS workloads)。由于以前I/O带宽的限制，Andy等人相信这个问题没有很好地被注意到

主要存在三个瓶颈：
1. 页表征用

2. OS单线程的页面回收

3. TLB shootdowns

前两个问题可以通过调整OS来缓解，然而TLB的问题更加棘手。前文提到，TLB shootdowns发生在一个core需要向其他cores广播其TLB条目无效时，由于不同核心之间TLB需要进行同步，带来了昂贵的开销。解决这一问题既需要对处理器微架构的修改，也需要对OS kernel进行调整(这不是你我能实现的🙈)

## 实验分析

论文的最后一部分，是通过实验来验证使用`mmap`的系统中存在性能问题

### 实验环境

- AMD EPYC 7713 processor(64 cores, 128 hardware threads)

- RAM: 512 GB (100GB用于OS kernel page cache)
- Persistent storage: 10 * 3.8 TB Samsung PM1733 SSDs(7000 MB/s的读吞吐量, 3800MB/s的写吞吐量)
- 作为对比，使用`fio`并通过直接I/O(`O_DIRECT`)来绕过OS的page cache

实验主要集中在只读工作负载，由于写操作需要额外的保证，因此会引入新的开销，无法准确控制变量。

-----

### Random Reads

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/random_reads.png" style="zoom: 67%;" />

working set size: 2TB(由于OS的cache大小只有100GB，因此95%的访问都会引起缺页异常)

由上图可以发现，使用`fio`时，带宽大约900K每秒。换句话说，使用`fio`几乎能使NVMe SSD的性能达到最大化

对于`mmap`，即使使用`MADV_RND`提示工作负载为random时，在大约27秒后带宽突然下降至0，然后缓慢恢复至大约`fio`的一般。突然下降是因为此时OS 的page cache已满，强行迫使OS从内存中剔除page所致。

最后，在上文提到，TLB的刷新与同步以及系统只有一个单线程进行页面回收会带来严重的开销。因此TLB shootdowns, 页表同步等将会严重降低性能。由上图可知，使用`fio`的TLB shootdowns十分稳定，而所有的使用`mmap`的系统TLB Shootdowns次数远高于使用file I/O的次数。

### Sequential Reads

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/sequential_reads.png" style="zoom: 67%;" />

对于sequential scan，可以看到使用file I/O与`mmap`也存在巨大的性能差异，特别是当使用到多个SSD时。通过实验可以发现，`mmap`只在单SSD，强制未命中阶段工作良好; 一旦页面回收开始或多个SSD时，`mmap`的性能相比`fio`要差2-20倍! 而随着SSD的性能的发展，`mmap`的表现将无法匹配传统的文件I/O性能。

## Conclusion

总而言之，当你考虑以下因素时，你不应该选择`mmap`来作为buffer pool的替代品：

1. 当你需要保证transaction的ACID属性时(上文的transactional safety)
2. 当你在触发缺页异常时，不希望被I/O阻塞(换言之，async I/O)或你希望手动控制哪些内容位于内存中
3. 当你需要关心错误处理时
4. 当你拥有SSD时，希望充分利用其吞吐量

如果你的工作集或者整个数据库可以完整地载入内存，并且工作负载为只读时，也许你可以考虑一下`mmap`。当然，如果现在正处于Database的风口，而你无需关心什么isolation level, consistency, 日后的维护，急需一款产品上市去吸引投资(~~骗钱~~)，那么这个时候你也可以考虑一下`mmap`
