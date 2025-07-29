+++
title = "Filter rules everything around me: Parquet中的Bloom Filter"
date = 2024-09-27
draft = false
author = "AsukaMilet"
tags = ["Database", "Apache Parquet"]
categories = "Database"
description = "Filter系列第一篇，Apache Parquet中的Bloom Filter"

+++

{{< katex >}}

对数据库有基本了解的都知道，Bloom Filter在数据库领域的应用非常广泛。RocksDB, Cassandra等使用Bloom Filter来避免搜寻位于磁盘上的SST，减少不必要的IO。Hash Join使用filter来过滤掉不匹配的key。除此之外，Bloom Filter也在网络，搜索引擎等领域有大量的应用，不过与本文无关。在本篇文章中，我们将介绍Parquet中的Split Block Bloom Filter的设计与实现。

----

## Background

让我们首先对传统的Bloom Filter做一个简单的回顾。原始的bloom filter使用一个bitmap来存储信息。当一个key要插入时，我们使用\(k\)个hash function将给定的key映射到bitmap对应的位置上，并将其设置为1。对于查询，我们使用相同的\(k\)个hash function得到\(k\)个位置，如果这些位置中至少有一个bit被置为0，那么说明给定的键一定不在filter中。由于bitmap中的bit可能被其他键设置为1，因此bloom filter存在一定的假阳率(*false positive rate*)，或者我们称之为误报率。对于Bloom Filter的假阳率，我们可以使用以下公式进行计算:
$$
f_p = (1-e^{-k\cdot n/m})^k
$$
\(f_p\)代表假阳率，\(k\)为hash function的数量，\(n\)为插入bloom filter的元素数目, \(m\)为整个bloom filter需要多少bits

因此当我们给定假阳率时，经过一定的数学变换，我们就可以得到计算为了让bloom filter达到预期假阳率时需要多少bits的公式:
$$
m = \frac{-k\cdot n}{ln(1-\sqrt[k]{1/k})}
$$

随着bloom filter中插入元素的增加，第一个公式的误报率也会增加，因为过滤器中会有更多比特被设置为 1。如果插入的元素太多，bloom filter就会变得毫无用处：无论是否已插入元素，它都会对任何查询报告出阳性结果。 因此，bloom filter在分配时需要提前知道数据集的大小，以保证向用户提供给定的误报率。除此之外，bloom filter也并不支持删除操作，也就是说，我们无法将已插入的元素从bloom filter中删除。

## Parquet中的Bloom Filter

截至本文编写时，Parquet有且仅有一种Bloom Filter实现，名为**Split Block Bloom Filter**，并且Parquet可以通过bloom filter来支持谓词下推。Parquet的每个column都有一个bloom filter, 并带有一个header来存储metadata。对于Bloom filter如何在Parquet中进行序列化并非本文重点，如果感兴趣的话可以自己参阅Parquet的文档。下面让我们来重点介绍Parquet使用的bloom filter本身的工作原理。

### Split Block Bloom Filter

Split Block Bloom Filter(SBBF)被分割为相同大小的block, block的数量至少为1，最多不高于\(2^{31}\)。其中每个block的大小固定为256bits，即32字节。而每个block又由8个连续的"words"组成，每个word都是32bits, 我们视为一个bit数组，其中每个bit要么被设置为1，要么被设置为0。因此我们可以将Block和SBBF定义为如下格式:

``` rust
struct Block([u32; 8]);

struct SBBloom(Vec<Block>);
```

接下来我们自顶向下的介绍SBBF的工作原理，从SBBF开始，到Block结束。

#### Split Block Bloom Filter本身支持的操作

对于SBBF，最主要的就是两个操作: `insert` 和 `might_contain` 。对于每次调用，SBBF只对一个block执行相关操作，`insert` 和 `might_contain` 两者接受一个 `u64` 的hash value作为参数。在对block执行相关操作之前，我们首先需要找到对应的block。SBBF使用hash的最高32bits来寻找对应的block，方法如下:

``` rust
fn block_index(&self, hash: u64) -> usize {
   (((hash >> 32).saturating_mul(self.0.len() as u64)) >> 32) as usize
}
```

我们使用hash的最高32bits和SBBF中的block number数量相乘，之后再使用结果的最高32bits作为block index。这个方法避免了昂贵的求模操作，对于该方法的原始出处，可以参见: [Efficient Hash Probes on Modern Processors](https://dominoweb.draco.res.ibm.com/reports/rc24100.pdf)

在得到对应的block index之后，我们使用hash function的最低32bits作为参数，传递给block相关的函数，对于一个已经构建完成的SBBF工作就结束了。除此之外，在构建SBBF时我们需要计算block number的数量，前文提到，对于预期的误报率，我们可以根据相关公式计算出bloom filter需要多少bits：
``` rust
pub fn number_of_bits(entries_num: usize, false_positive_rate: f64) -> usize {
    assert!(
        (0.0..=1.0).contains(&false_positive_rate),
        "false positive rate must between 0.0 and 1.0"
    );
    // For Split Block Bloom Filter, always use 8 hash functions for SIMD acceleration
    let num_bits = (-8.0 * entries_num as f64) / (1.0 - false_positive_rate.powf(1.0 / 8.0)).ln();
    num_bits as usize
}
```

在得到需要的bits之后，我们需要将bits转换为block的大小，算法很简单，`bits/8` 然后对2进行对齐即可。例如:
`bits / 8` 是49bytes, 那么block的总大小就是64bytes。如果 `bits / 8` 的结果为128bytes, 那么block的总大小就是128bytes。除此之外，我们不要忘记每个block最小为32bytes, 数量范围位于\([1, 2^{31})\) 

#### 进入Block

与SBBF类似，block本身最重要的也就是两个操作: `block_insert`  和 `block_check` 。两者均接受一个32bits的hash值作为参数，上文提到过，这32bits的值就是SBBF接受的hash值得最低32bits。

如果你对标准bloom filter有一定了解的话，你应该知道我们需要\(k\)个hash映射来设置bitmap中的值，SBBF也不例外，但是在SBBF中我们并不直接使用不同的hash function, 而是使用一个名为`mask` 的操作，`mask` 接受传递给block的hash值，并返回一个 `Block`，对于这个返回值，应该满足以下条件: block中的每个word有且仅有一位被设置为1。SBBF默认使用8个映射值来更好地获得SIMD加速，因此 `mask` 的作用就是传统的bloom filter的映射并设置bitmap的bit。

``` rust
const SALT: [u32; 8] = [
    0x47b6137b_u32,
    0x44974d91_u32,
    0x8824ad5b_u32,
    0xa2b7289d_u32,
    0x705495c7_u32,
    0x2df1424b_u32,
    0x9efc4947_u32,
    0x5c6bfb31_u32,
];

fn mask(x: u32) -> Block {
        let mut result = [0_u32; 8];
        for i in 0..8 {
            result[i] = x.wrapping_mul(SALT[i]);
        }
        for i in 0..8 {
            // Value of `val` is in range [0, 31]
            result[i] >>= 27
        }
        for i in 0..8 {
            result[i] = 1 << result[i];
        }
        Block(result)
    }
```

`block_insert`要做的事情不过是将`mask`返回结果中的bit一一对应地设置在自己的bitmap中，而`block_check`要做的操作就是一一检查，如果某个bit没有被设置说明给定元素一定不在filter内，否则可能存在于filter。

``` rust
fn insert(&mut self, hash: u32) {
   let mask = Block::mask(hash);
   for i in 0..8 {
       self[i] |= mask[i];
    }
}

fn check(&self, hash: u32) -> bool {
    let mask = Block::mask(hash);
    for i in 0..8 {
        if self[i] & mask[i] == 0 {
            return false;
        }
    }
    true
}
```

## Conclusion

Split Block Bloom Filter的想法与实现并不难，细心的话可以发现，每个block就相当于一个标准的bloom filter。与标准的bloom filter相比，split block bloom filter对缓存更友好，例如，我们原来有一个5MB大小的标准bloom filter就无法很好地装入cache之中，而split block bloom filter每次只需要操纵32bytes大小的block。

当然，相对于传统的bloom filter，split block bloom filter固定使用8个映射值，牺牲了一定的误报率，但是相对的是SIMD加速带来的高速构建与查询。

除此之外，split block bloom filter由于默认使用32bytes的block和对齐要求带来了新的空间开销，我们可以想办法减少每个block所需要的大小，但那就是另一篇blog的内容了。

对于Split Block Bloom Filter的完整代码，可以在这里找到: [Rust Playground](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=74b232805018d7a30f21b1b590f2c39c)

## Reference

1. [Apache Parquet Bloom Filter](https://parquet.apache.org/docs/file-format/bloomfilter/)
2. [Parquet-rs Bloom Filter implementation](https://github.com/apache/arrow-rs/blob/master/parquet/src/bloom_filter/mod.rs)
3. [Parquet Java Bloom Filter implementation](https://github.com/apache/parquet-java/blob/master/parquet-column/src/main/java/org/apache/parquet/column/values/bloomfilter/BlockSplitBloomFilter.java#L225)
4. 关于Split Block Bloom Filter的空间开销，InfluxDB有一篇很好的[Blog](https://www.influxdata.com/blog/using-parquets-bloom-filters/)。除此之外，他们还有一个相关的分析工具[parquet-bloom-filter-analysis](https://github.com/influxdata/parquet-bloom-filter-analysis)
5. [Split block bloom filters](https://arxiv.org/pdf/2101.01719)
6. 对于`mask`使用的计算映射值的方法，灵感来自于[A Reliable Randomized Algorithm for the Closest-Pair Problem](http://hjemmesider.diku.dk/~jyrki/Paper/CP-11.4.1997.pdf)

