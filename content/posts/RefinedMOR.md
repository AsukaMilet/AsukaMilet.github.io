+++

title = "Benchmarking, Analyzing, and Optimizing Write Amplification of Partial Compaction in RocksDB"
date = 2025-01-03
draft = false
author = "AsukaMilet"
tags = ["Database", "LSM", "Paper"]
categories = "Database"
description = "Analysis about leveled compaction file picking policy"

+++

{{< katex >}}

I recently read ‘[Benchmarking, Analyzing, and Optimizing Write Amplification of Partial Compaction in RocksDB](https://openproceedings.org/2025/conf/edbt/paper-114.pdf)’ by Boston DISC lab published at EDBT25. In this article they analysed the existing file picking policy based on RocksDB and explored the potential room for optimization, I felt rewarded after reading it so I chose to record some of the content that I thought valuable.

This article can be seen as a further study of the paper which they published at [VLDB21](https://vldb.org/pvldb/vol14/p2216-sarkar.pdf), and if you haven't read that article yet, I recommend it.

I'm going to assume you have a basic understanding of LSM before reading this article, so we won't go into detail about what compaction is, etc.

## Background

Classical leveling and tiering LSM employ full compaction which all data in a level has to be involved in one compaction task. When larger levels are compacted, the compaction takes longer time to complete, and there could be a temporarily high spike for storage space consumption. Besides, we also suffer from compaction latency.

To alleviate the above problem, a widely used optimization in leveling LSM is to divide a sorted run into multiple smaller partitions. We use the term *SSTable* to denote such a partition later in the text. This optimization breaks large compaction into multiple smaller ones, bounding the processing time of each compaction as well as the temporary disk space needed to create new SSTables. 

By partition, each compaction in a leveling LSM just needs to pick one SST[^1] in the level \(i\) to merge with SSTs in level \(i + 1\) that have overlapping key ranges. For sequentially created keys, essentially no merge is performed since there are no SSTables with overlapping key ranges. For skewed updates, the merge frequency of the SSTables with cold update ranges can be greatly reduced.

## File Picking Policies

File-picking policies control which SST of a level to compact. There are currently four strategies in RocksDB:

1. RoundRobin(RR) picks a file in a level in a round-robin manner in the key space.
2. MinOverlappingRatio(MOR) picks the file that has the minimum overlapping ratio. The overlapping ratio of a file in level 𝑖 is defined as the fraction between the total bytes of files in level 𝑖 + 1 that overlap with this file and the size of that file[^2]. 
3. OldestLargestSeqFirst(OLSF) picks the file of which the latest key-value pair is the oldest. OLSF is also termed as the coldest picking policy.
4. OldestSmallestSeqFirst(OSSF) picks the file of which the smallest sequence number is the oldest among all other files in the same level. OSSF is also called the oldest file-picking policy.

In addition to this, RocksDB support file picking policy based on [the number of tombstones](https://github.com/facebook/rocksdb/blob/d6c838f1e130d8860407bc771fa6d4ac238859ba/include/rocksdb/options.h#L85), and [Lethe](https://dl.acm.org/doi/10.1145/3318464.3389757) has implemented a file picking policy based on lifetime of tombstones, but these policies are beyond the scope of this article, and we will focus on the four policies mentioned above.

<p align="center"><img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE%202025-01-02%20144400.png"></p>

### New RoundRobin in RocksDB

The above figure shows the classic RR policy. However, according to the paper, classic RR actually fail to compact the key space in a round-robin manner(unfortunately, I haven't actually used the RR policy myself). They claim that RocksDB re-implements RR via a splitting mechanism which enforces every compaction that outputs in level \(i\) has to split the file based on the existing cursor of level \(i\)[^3].

In the new implementation, after a compaction from level \(i\) completes, the cursor in level \(i\) moves to **the smallest key of the successive file of the picked file in the last compaction**, and the next compaction from level \(i\) will pick the first file (from left to right) that is **larger than or equal to** the cursor. By skipping the cursor past the gap between the last picked file and its successor, new keys arriving into that gap in level 𝑖 will not be compacted immediately. 

<p align="center"><img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/roundrobin_in_rocksdb.png"></p>

For example, as shown in Figure 3, the first compaction moves the cursor to <key, 6>, then a compaction from L0 triggers a L1 compaction. In this L1 compaction, the sstable starting with <key, 6> will be chosen and thus key at L1 can be uniformly compacted in a round-robin manner. 

The newest version of RR allows multiple consecutive files to be selected together, as long as the overall bytes involved in this compaction do not exceed a user defined threshold. When two or more files are selected, the cursor still changes into the smallest key of the successive file of the last picked file.

### Aligning Compaction Output File Boundaries

Since RocksDB 7.80, RocksDB has introduced a new optimization to reduce write amplification for leveled compaction. I personally thought RocksDB's [Blog](https://rocksdb.org/blog/2022/10/31/align-compaction-output-file.html) on this part a bit hard to understand, this optimization is mentioned in this paper and their description is a bit more understandable to me.

In RocksDB, newly generated SSTables are not restricted to be exactly as large as the user-defined target sstable size. Instead, they can vary to align new sstables with the key boundaries in a grandparent level, i.e. level \(i + 1 + 1\), for smaller WA in future compactions to the grandparent level. This optimization does not apply to files generated at the deepest level as there is no grandparent level. Thus, the file size at the deepest level should be exactly the same as the target one. 

The new mechanism works as follows:

<p align="center"><img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/boundary_rocksdb.png"></p>

If we have a `SSTableBuilder` in current compaction, there are three conditions when RocksDB stops appending new entries:

1. When the builder size reaches the user-defined maximum compaction bytes or `2 * target_sst_size`. In this case, newly generated ssts never cross any grandparent boundaries.

2. When the next key crosses a grandparent boundary and the builder size reaches a threshold(less than the target sstable size). The threshold is calculated by:

   ```c++
   current_size > ((target_sst_size + 99) / 100) * (50 + min(num * 5, 40))
   ```

   `num` is the number of accumulated crossed boundaries. And the minimum value is 1.

   Let's do some simple maths by taking the value of `num` to 1 and we'll round 55 to 50, so we have:

   ```c++
   current_size > ((target_sst_size + 99) / 2
   ```

   If we ignore 99 then we can roughly estimate the minimum size of the sstable to be around `target_sst_size / 2`.

   This condition allows the newly generated sstables to overlap with fewer files in the grandparent level.

3. When the next key crosses more than two grandparent boundaries and the current builder is larger than `target_sst_size / 8`. Close the builder before the next key avoids producing files that overlap a skippable file in the grandparent level.

## Benchmarking file picking policies

People from Boston have done rich benchmarking of file picking policies and have validated some of the existing observations and made some new ones, and I'll pick out what I think are interesting observations to discuss here.

### File-picking policies have minor impact on WA for update-intensive workloads

<p align="center"><img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/update__file_picking_wa.png"></p>

When update proportion is large, all four file-picking policies have lower and similar WA according to their benchmark. They analysed the relevant reason and concluded that with increasing updates, entries with duplicate keys can be updated in memtable flushes and L0 compactions.

### RR leads to a lower average WA than MOR when updates exhibit a higher skew

They found the average WA of RR decreases faster than MOR when the update distribution becomes more skewed(consider that you have some keys that get updated frequently). They think this benefit of RR because frequently updated keys stay longer in L1. Since RR waits for the cursor to iterate over the whole key range, MOR has a higher probability of compacting the most frequently updated key to deeper levels than RR, and leads to a higher WA when updates have higher skew.

### MOR scales better

They further scaled the workload to test the performance of different file picking policies under larger workloads and they found that:

> MOR scales better than other policies by exhibiting lower average WA and higher stability for uniform update distribution

<p align="center"><img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/scale_file_policies.png"></p>

So when you use leveled compaction, and don't know what file picking policy to use, start with MOR.

## Exploring the space for optimization

After a wealth of benchmarking, they attempted to further investigate whether they could do better than what was already in place. They used a very brute force approach: enumeration, they narrowed down the workload, and then kept enumerating to try to find the optimal choice(minimum WA), and then compared the best choice to the above four options to find the potential room for optimization.

Then they found an interesting situation: the difference in WA between RR and MOR and the optimal choice is often not very large, but the optimal choice behaves more stably.

<p align="center"><img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/optimal_wa.png"></p>

### Why this instability exists?

After they noticed this phenomenon, they conducted further research on MOR, and they found that at each compaction, there may be multiple sstables where the gap in overlapping ratio is not significant, perhaps less than 1%. However, MOR strictly selects the minimum overlapping ratio, MOR can lead to unstable WA for the same workload. They summarised the following conclusion:

> Picking different files that have similar WA (overlapping ratio) in the current compaction can lead to substantially different final WA.

If you think about it a little bit, you'll see that it's very similar to the greedy algorithm, where we choose the current local optimal solution at each compaction(minimum overlapping ratio), yet in the long term we don't reach the overall optimal.

They use an example to explain this issue.

<p align="center"><img src="https://raw.githubusercontent.com/AsukaMilet/BlogPhotos/master/Typora/mor_disadvantage.png"></p>

In the above figure, L1 has four sstables with the same overlapping ratio, all 1.0. RocksDB will sort the sstables by overlapping ratio, and when the *scores*(RocksDB also considers TTL to calculate scores, we don't consider TTL here and only care about the overlapping ratio) are the same, RocksDB will pick the sstable with the smallest key for compaction to make the algorithm more deterministic[^2].

But what happens when we consider MVCC? For example, the upper figure becomes:

```rust
L1: [1..10] [11v1..11v_n] [11v_(n+1)..30] [31..40]
L2: [1..10] [11v1..11v_n] [11v_(n+1)..30] [31..40]
```

[11v1..11vn], [11v_(n+1)..30] have similar probabilities of being selected.

In their paper, they analyzed in detail why this randomness leads to WA instability.

### RefinedMOR

Based on the above analysis, they claim that when there exist multiple sstables with similar overlapping ratios, we should not strictly choose the one with the smallest overlapping ratio, but should consider all these sstables as candidates.

The subsequent process is as follows:

1. Select sstable with minimum overlapping ratio
2. Based on the minimum overlapping ratio, we use the following formula to calculate threshold

```rust
u64 threshold = (1 + th) * min_overlapping_ratio
```

3. Iterate over all sstables at the current level and collect all sstables with overlapping ratio less than threshold
4. If there is only one sstable in the candidates, it means that this sstable is the one with minimum overlapping ratio, and we choose it directly
5. If there is an sstable that is the last sstable of the current level, we select it directly
6. Select the sstable whose next sstable in the level has the largest overlapping ratio

```rust
let mut min_combined_ratio = u64::MAX;
let mut select_idx = 0;
// `idx` is the index of the eligible sstable in the original level
candidates.iter().for_each(|idx| {
   let gap = (overlapping_ratios[idx] as f64) * 0.05 - overlapping_ratios[idx + 1] as f64;
    if gap < min_combined_ratio as f64 {
        min_combined_ratio = gap as u64;
        select_idx = *idx;
    }
});
```

Next, I will give my understanding of why RefinedMOR can solve the problem of WA instability.

1. The last SSTable of the original level is preferred. This way, we will not trigger boundary alignment when generating a new SSTable (because there is no SSTable for us to use). So we can maintain the overlap ratio between the newly generated files and the lower layer in a relatively ideal state.

2. Then we tend to select the file which the subsequent SSTable has the largest overlapping ratio. Based on my understanding, due to the existence of the align mechanism, we need to ingest the data of the subsequent SSTable when generating a new SSTable. Through the above selection scheme, we will have a high probability of triggering early truncation, which will help reduce the overlapping ratio of the subsequent SSTable. In this way, when this subsequent SSTable is selected, WA will be reduced.

There are a few notable points here

- When calculating `overlapping_ratio` and `threshold`, our type definition is `u64`. So due to truncation, according to the MOR formula previously mentioned, `threshold` may be the same as `minimum_overlapping_ratio`, e.g. both are 1. For this issue, RocksDB will multiply the numerator by 1024 when calculating the `overlapping_ratio`.
- For `th` in `threshold = (1 + th) * min_overlapping_ratio`, authors choose 0.05. I emailed to ask if they measured any other numbers and they didn't look into it further here, it's not an empirically optimal number. But they also think larger `threshold` should be fine.
- I initially thought we could start by collecting the sstables that have the smallest overlapping ratio by using something like [Itertools](https://docs.rs/itertools/latest/itertools/trait.Itertools.html#method.min_set_by). This idea is not correct though, if we follow this pattern we are back to the strictly select of minimum overlapping ratio and exhibits randomness described above.

## Conclusion

There isn't much research on file picking policy in LSM, and this paper does a good job of filling in the gaps. Besides that, they mention RefinedMOR as a very rudimentary improvement, but I can't think of how to further improve file picking policy while keeping the implementation simple. Feel free to email me if you have any great ideas. I hope this article will be helpful for you.

## Reference

[1] [Option of Compaction Priority](https://rocksdb.org/blog/2016/01/29/compaction_pri.html)

[2] [Benchmarking, Analyzing, and Optimizing Write Amplification of Partial Compaction in RocksDB](https://disc.bu.edu/papers/benchmarking-analyzing-and-optimizing-wa-of-partial-compaction-in-rocksdb)

[3] [LSM-based storage techniques: a survey](https://dl.acm.org/doi/10.1007/s00778-019-00555-y)

[4] https://github.com/BU-DiSC/rocksdb-for-partial-compaction-analysis

[^1]: To be precise, this depends on the file picking policy, details can be found in [RocksDB PickFileToCompact](https://github.com/facebook/rocksdb/blob/3570e4f5ffb29bc21b9afb388104a0a04f9af356/db/compaction/compaction_picker_level.cc#L95).
[^2]: [RocksDB SortFileByOverlappingRatio](https://github.com/facebook/rocksdb/blob/3570e4f5ffb29bc21b9afb388104a0a04f9af356/db/version_set.cc#L3941)
[^3]:  [RoundRobin in RocksDB](https://github.com/facebook/rocksdb/blob/b75438f9860e3cff5e713917ed22e0ac394a758c/include/rocksdb/advanced_options.h#L61)

