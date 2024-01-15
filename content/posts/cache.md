+++
title = 'What Every Programmer Should Know About Memory: Cache篇'
date = 2023-09-27
draft = false
author = "AsukaMilet"
tags = ["Architecture", "Cache", "Paper"]
categories = "Paper"
description = "来自于一篇有点年头却又不失经典的paper"
math = true

+++



最近终于将*What Every Programmer Should Know About Memory*这篇paper给读完了，这篇发表于2007年的paper洋洋洒洒写了一百多面，在上面花费的时间大大超出了我的预期。鉴于原文的篇幅之长，决定将读完之后的感悟拆分成2篇文章，前一篇即本文重点放在Architecture以及Cache的原理上，后一篇将其放在"What programmers can do"这一作者花费了大量笔墨叙述的部分 ~~(什么时候？希望不会拖太久吧)~~

## 从已被弃用的南北桥谈起😅

在NorthBridge和NorthBridge的时代，所有的CPU处理器通过一根通用总线 (*Front Side Bus*)与NorthBridge相连。NorthBridge包含memory controller，负责与RAM进行交互。而CPU为了与其他的I/O设备进行交互，必须通过SouthBridge (因此SouthBridge也被称为I/O Bridge)。SouthBridge则通过各种各样的总线与I/O设备进行数据传输

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/N-S%20Bridge.png" style="zoom:67%;" />

通过上面的简单叙述，我们可以得出以下基本结论：

1. 任一CPU处理器与其他处理器进行交互时必须共享与NorthBridge交互的那根bus
2. 所有与RAM进行的交互都必须通过NorthBridge
3. RAM只有一个port(multi ports仅在特殊设备如路由器等出现，并非本文重点)
4. CPU与I/O设备之间的交互需要通过SouthBridge并由NorthBridge进行转发

### 南北桥架构的问题

在早期的PC架构中，I/O设备访问内存都需要通过CPU，极大地影响了性能。后来通过DMA (*Direct memory access*)解决了这一问题，DMA允许I/O设备在NorthBridge的帮助下直接load/store data in memory而无需经过CPU。但是DMA也存在自身的缺陷：DMA requests同时会导致与CPU的memory requests争抢NorthBridge带宽资源。

## NUMA Architecture

随着硬件技术的发展，传统的FSB已经被弃用，并且NorthBridge的功能被继承到CPU芯片之内。此时我们就来到了现在大部分计算机所采用的架构——NUMA (*Non-Uniform Memory Architecture*)

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/NUMA%20arch.png" style="zoom: 67%;" />

在NUMA中，Memory controllers被集成到CPU中，并且CPU直接与memory进行交互。直接优点就是避免了争抢NorthBridge带宽资源带来的性能限制。

但是NUMA也有它的缺陷，由于所有的CPU均需要具有访问所有memory的能力，由于NUMA架构的特点，此时memory对于CPU来说并非均匀分布。对于某个处理器来说，local memory(memory attached to the processor)的访问速度虽然不受影响(相对南北桥可能更快)，但是该处理器访问attached到其他处理器的memory就会产生影响。

以上述图示为例，\\( CPU_1 \\)要访问attached to \\( CPU_2 \\)的memory就需要经过一次内部数据传输(interconnect); 访问attached to \\(CPU_4\\) 的memory就需要经过两次内部数据传输。处理器之间的数据传输会导致额外的开销，这是NUMA架构带来的缺点。

## CPU Caches

在介绍完论文的前置部分，也就是关于架构的部分之后，我们现在来到了重点cache部分。关于cache的内容，许多Computer Architecture的相关书籍都有介绍，但是该paper关于cache的内容仍然十分经典，有不少值得细读之处。

对于CPU内的cache，主要由SRAM构成，完全由硬件而不是OS来控制，主要作用为维持DRAM(常说的内存)中容易被寄存器访问到的数据的副本。

### Caches in the Big Picture

在详细介绍cache的相关内容之前，首先从宏观的角度来叙述绝大部分PC中cache的构成：

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/Level%203%20Cache.png" style="zoom:67%;" />

<center>Single Core下的三级Cache
</center>

> 在上图中，L1i Cache代表level 1 instruction cache; L1d Cache代表level 1 data cache

在如今的计算机中，多核处理器已经成为常态，将Cache与上述的NUMA架构结合起来后如下：

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/NUMA_cache.png" style="zoom:67%;" />

<center>NUMA Architecture with Cache</center>

在上述图例中，存在两个处理器，每个处理器中存在两个核心(core)，每个core独享L1i和L1d cache，共享L2和L3级cache。

#### 为什么将指令和数据cache分开？

在memory中，code segment和存储数据的segments如 *.data, .bss*是分开的(感谢virtual memory和process!) 因此对于cache来说将两者分开能够工作地更好。

不仅如此，由于指令的解码过程对于绝大多数处理器是很慢的，因此缓存已经解码的指令有助于加速指令的执行过程，尤其是出现分支预测错误导致pipeline全部作废，处于empty状态时。

#### Cache如何工作？

在了解cache在计算机中的大致构成之后，现在可以继续叙述cache是如何工作，以及是如何提高计算机的性能了。

由于cache通常无法包含整个main memory的内容(如果允许我们还需要内存干什么呢🤡)，但是所有的memory address应该是可缓存的，因此每个cache条目使用main memory中的data word地址来进行定位。用于搜索cache的地址可以为virtual memory address或physical memory address，但在实现中通常使用physical memory address。

##### 如何搜寻Cache条目？

存储在cache中的条目并不是单一的word，取而代之的是"lines of several contiguous words"~~(实在想不到好的翻译了)~~。在paper写成的年代，每行cache条目的大小通常为64bytes。如果memory bus的字长为64bits，则填充每行cache条目需要8次copy，DDR可以更高效地支持该模式 ~~(为什么？不知道🙈)~~

以L1d cache为例，当寄存器需要某个给定位置的内存内容时首先搜索cache本身，如果在cache中存在所需内容则万事大吉，直接使用cache中的内容即可，如果不存在则需要将对应的内容从内存中copy至cache。

搜索的依据则为内存对应的memory address(如上所述，使用physical memory address)。以32-bit address为例，可以将memory address划分为以下几个部分：

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/Cache_format.png"  />

<center>Cache format</center>

- *Offset*: 在cache每行大小为 \\( 2^O\\)的情况下，低\\(O\\) bits被用于cache line内部的偏移量。换句话说，offset用于定位所需数据位于cache line的哪一个slot之内(slot大小通常为byte)
- *Cache Set*: \\(S\\) bits用于选择"cache set"，此时只需要理解许多cache lines被划分至同一个集合/组中即可，而\\(S\\) bits说明整个cache中存在\\(2^S\\)个集合/组
- *Tag*: 剩余的\\(32-S-O = T\\) bits作为标记位，作用为将同一个cache set中的不同cache line区分开来
----

当处理器要对内存执行写操作时，仍然要先将cache line中的内容加载并在寄存器中进行修改后写至cache中。被修改但还没有被写回内存的cache line称为"dirty"，通常在Cache format中需要一个额外的dirty flag说明是否为dirty。一旦被写回至内存中，dirty flag也随之复位。

> 此处可以额外了解[Cache inclusion policy](https://en.wikipedia.org/wiki/Cache_inclusion_policy)

#### Cache coherency

在NUMA架构中，所有的处理器都应该有能力访问整个内存，而引入cache之后带来的一个新的概念：缓存一致性(cache coherency)，为了确保缓存一致性，大致规则如下：

1. 一个dirty cache line的内容不会出现在其他处理器的cache中

例如：两个处理器同时访问内存中的同一个区域，在各自的L1d cache中保存了对应的副本。此时 \\(CPU_1\\)执行写操作产生了一个dirty cache line，但是这一操作不影响 \\(CPU_2\\)中的副本，\\(CPU_2\\)仍维持原来内存中对应区域的副本。

2. 同一cache line的干净副本可以驻留在任意的多个caches(允许多个读者同时存在)

如果这两条基本规则得以维护，在NUMA架构中所有的处理器就能高效地利用他们自己的cache。任意处理器需要做的工作就是监视其他处理器对内存发起的写操作，并将对应的地址与自身cache中缓存的cache line的对应位置进行比较，具体的细节在后文中详述。

最后用一张Memory hierarchy来作为Caches in the Big Picture的总结：

![](https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/memory_hierarchy.png)

> 还可以看看[Latency Numbers Every Programmer Should Know](https://colin-scott.github.io/personal_website/research/interactive_latency.html)

### 魔鬼隐藏在细节里😈

在概述了Cache的工作原理之后，现在可以深入Cache的实现细节，让我们从cache在计算机内部的具体实现开始。

#### Associativity

##### Fully Associative cache(全相联)

前面提到过，cache set这一部分用于判断待搜寻的cache line位于哪一个集合/组中，但是在具体的实施中可以将整个cache放在一个组中。此时对应的cache format如下：

![](https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/fully%20associative%20cache.png)

此时原来的cache set部分可以划给tag或block或者全部为0(tag的位数应该能够映射到整个physical memory address space)。处理器仅使用tag部分来进行cache line的匹配。

虽然全相联理解起来非常容易，但是在实际中并不太现实。例如存在一个大小为4MB的cache，每行的大小为64Bytes，则存在 \\(2^{16}\\) (65536)个条目，而CPU在进行匹配时需要将tag进行迭代匹配，在最糟的情况下可能需要匹配65536次！正是因为这个明显的缺陷，全相联通常只在容量极小的cache中实现(e.g 在某些Intel处理器中TLB的实现为全相联的)。

> 侧面反映出TLB的大小通常比L1d, L1i的大小还要小

##### Direct-Mapped Cache

对于*direct-mapped*形式，每一行自身就属于一个组，此时对应的cache format如下：

![](https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/Direct-mapped_cache.png)

而它的物理实现如下：

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/direct_mapped_cache_mecha.png" style="zoom: 80%;" />

因为每一行自身就属于一个组，因此用S作为输入来选择tag的哪一个输入传递给比较器。direct-mapped cache的优点就是非常快，并且易于实施，相对于全相联需要迭代匹配，direct-mapped cache只需要mux和comp两次运算即可。

但是direct-mapped cache有它自身的缺陷：只有当程序使用的地址空间相对于direct-mapped cache均匀分布时才能较好地工作。如果不满足这一条件(想想局部性，不满足这个条件非常容易)，一些cache条目将会被频繁地映射并且发生频繁地替换(最坏的情况可能需要写回内存)，而其他的条目几乎不会被访问，并且保持empty状态。

##### Set associative cache

在描述上述两种实现方式之后，现在让我们正式介绍组相联，组相联可以较好地解决上述两种实现的缺陷，此时cache format如下：

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/set_associative_cache.png" style="zoom: 75%;" />

在组相联实现中，极小部分的cache lines被划分到同一个组中，而同一个组的cache line使用tag来进行匹配。tag的匹配为并行执行，此时tag的匹配时间相对于全相联的匹配大大减少。

##### 关联性总结

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/cache_size_associativity.png" style="zoom: 80%;" />

由上图(working set size = 5.6MB)可以看出，对于每一行大小为32bytes的cache，增加关联性可以显著减少cache misses发生的次数。例如对于一个8MB大小的cache从direct mapping到2路组相联，缓存未命中发生的次数减少了几乎44%。

作者提到在某些文献中可以看到引入关联性的效果等同于将cache size扩大一倍，在某些情况下确实成立，但如果继续加倍关联性命题不成立。很明显，在上图中随着关联性的增加，缓存成功命中的次数只增长很小的一部分。

随着working set的增大，关联性能够减少缓存未命中的次数就越高。

通常情况下，对于单线程程序将关联性从8向上增加只会对性能带来微弱的提升。但随着超线程处理器的引入，多个核心共享L2级cache。假设此时有两个程序共同使用同一个cache，导致实际的关联性减半(对于4核处理器，减到1/4)。因此在理想情况下随着CPU核心的数量增加，共享cache的关联性也应该随之增加，但通常不太现实(16路组相联的实施就不是很常见了)，为了解决这个问题引入了L3级cache。

#### 写操作

对于user级别的代码而言，cache line与内存之间的数据不一致应该是不可见的，但对于kernel级别的代码而言，它有时需要控制cache flushes。换言之，如果一个cache line被修改了，在user级别的视角中与之对应的内存单元中的数据也被立即修改了，就好像不存在cache一样(registers与内存保持同步)。

为了实现这一幻象，有两种策略可以实施：

1. write-through cache implementation(直写)
2. write-back cache implementation(写回)

##### Write-through

Write-through cache是实现缓存一致性(cache coherency)的最简单的方法。如果cache line被修改，则处理器立即将修改后的cache line内容写到对应的内存中。这就确保了无论什么时候，内存和cache之间是同步的。

这一策略虽然易于实施，但对于性能而言并不是非常快。例如：某个程序会经常修改一个局部变量，如果采用直写策略则会频繁引发cache和内存之间的数据传输，尽管该变量很可能不会被其他部分所使用。

#####  Write-back

Write-back相对于write-through复杂得多。当某个cache line被修改时，处理器不会立即将被修改后的内容写至内存，取而代之的是仅仅将该cache line标记为dirty。当cache line需要发生替换时，dirty flag指示处理器将dirty cache line中的内容写至内存(**和buffer pool何其相似！**)。由于write-back可以显著提升性能，因此几乎所有的处理器都以这种方式实现cache。

-----

对于write-back策略，当有多个处理器/核心/hyper-thread时，必须确保不同的处理器访问内存上的同一个存储单元时，应该看到相同的内容。例如 \\(CPU_1\\) 某个cache line的dirty flag被置为1，\\(CPU_2\\) 访问对应的内存时不应该使用内存中的内容，取而代之应该使用\\(CPU_1\\) dirty cache line中的内容。对于具体的策略，将会在下文详述。

#### Multi-processor Support

在前面我们谈到了NUMA架构，谈到了缓存一致性，谈到了write-back策略存在的问题。现在，让我们将一切的一切放在本文放在最后的一个部分进行终结，让我们通过多处理器情况下如何保证缓存一致性来看看CPU设计者们是如何解决上述问题的。

以write-back策略部分提到的问题为例。在实际情景中，如果存在该问题，解决方案是将dirty cache line中的内容转移给所需的处理器。听上去非常简单，但是这又带来的新的问题，即：

> 一个处理器如何确定某个cache line在其他处理器的cache中被标记为dirty?

由于很多时候处理器对内存只是进行读操作，仅仅因为另一个处理器的cache加载了该内存单元中的内容就判断所需内容为dirty的假设是不靠谱的。不仅如此，因为写操作也很频繁(相对读操作可能没有那么频繁)，因此每对内存执行一次写操作就广播cache line的内容被修改了也不现实。

怎么办？现在让我们介绍......(戏剧性的暂停)......

##### MESI protocol

经过研究人员的不懈探索，为了确保缓存一致性，也是支持write-back策略最常用的机制称为 [MESI protocol](https://en.wikipedia.org/wiki/MESI_protocol)

在MESI协议中，一个cache line在某一时刻处于以下四种状态之一：

- Modified: the local processor修改了该cache line，与此同时该cache line中的内容并不存在于其他的cache中(e.g L1d cache in multi cores)

- Exclusive: 该cache line没有被修改但也没有被加载到其他处理器的cache中

- Shared: 该cache line没有被修改并且有可能存在于其他处理器的cache中

- Invalid: 该cache line无效(e.g unused)

可以将MESI协议的工作视为一个有限状态机，状态转移图如下：

<img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/MESI_trans.png" style="zoom:67%;" />

现在就让我们来读懂这张图：

1. 在起初，所有的cache lines均为空因此均处于 *Invalid* 状态

2. 之后，假设数据被加载至某cache line，此时cache line的状态取决于处理器执行的操作：

   - 如果处理器执行了写操作，则该cache line的状态转换为 *Modified*

   - 如果处理器执行的是读操作，此时cache line的状态又取决于其他处理器中是否加载了同一内存单元的内容。如果答案为是，那么该cache line转换为 *Shared*，否则为 *Exclusive*

3. 如果一个状态为 *Modified* 的cache line被local processor读/写，那么状态不会发生改变。如果是另一个处理器想要访问对应的内存单元，那么当前的处理器必须将处于 *Modified* 状态的cache line中的内容发送给发起请求的处理器，并且该cache line的内容转换为 *Shared*。不仅如此，发起请求的处理器应该将cache line的内容同步至内存之中，如果该操作没有完成，则状态转移不能发生

4. 如果另一个处理器在获取 dirty cache line的内容之后对其执行了写操作，那么最初的处理器应该将 cache line的状态标记为 *Invalid*(*Modified* \\(\rightarrow\\) *Shared* \\(\rightarrow\\) Invalid)。作者提到，这就是臭名昭著的"**请求所有权(Request For Ownership)**"操作，在cache中执行这一操作时是成本相对较高的

5. 如果某个cache line的状态为 *Shared*，那么local processor执行读操作不会引起状态转移。如果执行写操作，那么发起写操作的处理器中，该cache line的内容转变为 *Modified*，与此同时其他处理器中可能存在的副本应该被标记为 *Invalid*

6. *Exclusive*与*Shared*状态最关键的区别在于：local processor的写操作将不会被广播给其他处理器

---

通过上面对于状态转移的叙述我们可以推测出多处理器下特定操作的成本，此时除了缓存未命中的成本之外，我们还需要考虑 RFO消息的成本，无论何时，RFO消息都会带来额外的开销。

在两种情况下，RFO消息是必须的：

1. 一个线程从一个处理器迁移至另一个处理器，此时所有的cache lines都应该转移至新的处理器
2. 某个cache line被两个处理器所需要并触发了写操作。(即使是在相同处理器的不同核心之间，这也是必须的，只是成本相对较低)

对于MESI协议，直到计算机中所有的处理器都对收到的消息进行回复之后才能发生状态转移，这也带来了一个悲伤的事实：最长的回复时间决定了MESI协议的速度，这也意味着MESI协议的性能瓶颈😭。在NUMA架构中，这个延迟能够变得非常高昂(当然，~~一切都是比较而言~~，和I/O肯定还是不能比)，并且会占用总线巨大的流量，因此尽量减少RFO消息总是好的。

因此对于程序员来说，应该尽量减少不同处理器/核心对同一内存单元的访问。

## Summary

恭喜你坚持看到这里，在原文中还有关于Cache需要发生替换时的策略(LRU)，缓存未命中带来的性能下降的详细分析，由于篇幅原因，不再赘述。如果你这对这些内容感兴趣，可以去查阅原文，细细品读。

## Reference

1. Ulrich Drepper. "What Every Programmer Should Know About Memory". November, 2007.
2. Randal E. Bryant and David R. O’Hallaron. *Computer Systems: A Programmer’s Perspective, 3rd edition*

3. [一篇论文讲透Cache优化 by 知乎—红星闪闪](https://zhuanlan.zhihu.com/p/608663298)
