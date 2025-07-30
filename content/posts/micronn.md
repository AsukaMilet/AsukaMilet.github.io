+++

title = "MicroNN: An On-device Disk-resident Updatable Vector Database"
date = 2025-07-29
draft = false
author = "AsukaMilet"
description = "How to implement vector search on the edge devices"

+++

最近正在研究Vector Database。为了加深理解，阅读了苹果在[SIGMOD25](https://dl.acm.org/doi/10.1145/3722212.3724444)发表的论文，介绍了如何在边缘设备上实现高效且可更新的基于磁盘的向量搜索。这篇论文叙事十分工程化，通俗易懂，即使不需要很多前置知识也能读懂(在这里我必须要吐槽一下很多VecDB相关的论文，通篇都是各种数学分析，根本就不是写给DB人的嘛，更像是炫耀自己数学有多好)。论文详细阐述了在资源受限环境下，如何设计和优化向量搜索系统，使其既具备高性能，又能满足实际应用需求。读完后，我受到了很多启发，决定将有价值的内容记录下来。

## 边缘设备实现向量搜索的挑战
论文提出，在边缘设备上实现向量搜索的核心需求有以下几点:
1. **资源受限**: 边缘设备通常具有各种各样的环境，实现向量索引和搜索等功能时必须要考虑兼容性，能耗等问题。
2. **多租户**：边缘设备上的硬件资源通常是共享的，这就导致内存，算力等资源有限。与此同时由于后台管理，索引等在应用非活跃的情况下不应该驻留在内存中，这就要求索引和数据能够高效地存储在磁盘上。
3. **IO高效**: 由于索引和数据驻留在磁盘上，IO性能直接影响到向量搜索的效率。因此应该要保证IO的高效以及尽量减少IO，避免磨损闪存等导致降低设备寿命的问题。

基于上述挑战，苹果他们实现了一个可以在边缘设备上运行的名为MicroNN的向量搜索库(**并不是完整的向量数据库**)，特色如下:
- **基于磁盘的向量索引和ANN搜索**: 根据他们的测试，在百万级向量数据集上，MicroNN可以在7ms内完成查询并只使用大约10MB的内存。
- **支持混合查询**: MicroNN支持在同一个查询中同时执行向量搜索以及用户自定义的参数进行过滤。例如当用户搜索"Karina Drama"相关的照片时，我们可以用"Drama"作为过滤条件，提前筛选掉不属于"Drama"这首回归曲时期的照片。这样不仅能够加速搜索速度，还能提高搜索结果的准确性。
- **支持增量更新**: MicroNN支持对向量索引进行增量更新，减少重建整个向量索引的次数。
- **批量查询**: MicroNN支持批量查询，允许用户一次性查询多个向量，提高查询效率。
- **更好的一致性支持**: 通过基于SQLite的ACID支持，MicroNN具有更好的一致性，持久性和隔离性。

## 当前方案的缺陷
MicroNN提到了目前主流方案的一些缺陷
### 未考虑边缘设备的资源受限
目前主流的向量索引实现可以分为两派[^1]: 以HNSW为代表的基于图的索引，[DuckDB](https://github.com/duckdb/duckdb-vss), [Redis](https://redis.io/docs/latest/develop/ai/search-and-query/vectors/#hnsw-index), [Qdrant](https://qdrant.tech/documentation/concepts/indexing/)等就是基于HNSW。另一派是以IVF为代表的基于分区的索引，例如[LanceDB](https://lancedb.github.io/lancedb/concepts/index_ivfpq/), [FAISS](https://github.com/facebookresearch/faiss/wiki/Faiss-indexes)。然而这些索引大部分都是针对云端部署而设计，几乎都是纯内存索引并需要大量的内存。DiskANN是唯一一个针对SSD进行了优化的索引实现，但是它仍然需要将所有的向量压缩后存储在内存中，并不适合边缘设备[^2]。
### 对于写操作不友好
当下的向量索引实现大多针对只读工作负载进行了设计，HNSW等基于图的索引在更新时由于随机访问，需要大量的计算资源。IVF等基于分区的索引虽然更新相对友好，但是随着vector的增删和修改，centroid会慢慢不再反应cluster的实际情况，导致查询结果不准确。每次执行写操作都重新进行re-clustering又会消耗大量的资源和时间，这并不符合上述提到的需求。除此之外IVF等索引随着写操作的累积，还可能有partition不平衡的问题，导致查询性能下降。
### 不支持混合查询
上述提到，真实的业务场景中并不是纯粹的向量查询，通常还会有一些其他的过滤条件，例如时间，地点等。当前的向量索引实现大多不支持混合查询，或者实现非常简陋，只支持简单的数值比较，完整文本匹配等。
### 批量查询优化不足
批量查询是向量搜索中一个重要的优化点，当前的向量索引实现大多只支持单个向量查询。基于图的索引由于图的遍历问题，与生俱来的就对批量查询不友好，作者指出HNSWlib, DiskANN等目前的批量查询都只是简单地将多个查询串行化处理，无法充分利用批量查询的优势。而基于分区的IVF索引则在批量查询上有很大的潜力。

## MicroNN如何解决上述问题？
### 持久化功能
这里是我觉得MicroNN最有意思的地方，他们没有想方设法去把索引本身序列化，而是直接使用了SQLite，通过建表的方式，把索引和向量等映射至SQLite中。根据他们的说法，MicroNN定位是一个中间件，而不是一个从头实现的向量数据库。MicroNN通过SQL来和SQLite进行交互，根据我自己的理解，他们大概率自定义了一套API，而不是修改parser等来让SQL原生支持向量搜索。这样的好处有很多:
- **兼容性**: Lance使用了自己的存储格式，而许多其他的向量数据库都会有自己的vector type，序列化方式等，这就导致了向量数据库之间的互操作性很差[^3]。
- **ACID**: SQLite本身的并发控制允许他们直接实现单写多读，而他们使用了WAL模式可以无痛获得持久化，隔离性等特性。
- **工作量小**: 他们不需要从头实现数据库所有的功能，也不需要修改数据库内核组件如parser, type system等，大大减少了工作量(我给Turso修改planner等组件时那绝对是地狱般的体验。顺带一提，如果Turso的planner都是抄的SQLite的话，那我只能说SQLite的planner也是一坨)。

[^1]: [Bang for the Buck: Vector Search on Cloud CPUs](https://ir.cwi.nl/pub/35187/35187.pdf)

[^2]: Turso实现了DiskANN的一个变种，称为[LM-DiskANN](https://cse.unl.edu/~yu/homepage/publications/paper/2023.LM-DiskANN-Low%20Memory%20Footprint%20in%20Disk-Native%20Dynamic%20Graph-Based%20ANN%20Indexing.pdf)，貌似针对DiskANN进行了内存空间优化，该索引是基于图的，但是我暂时没有阅读相关论文

[^3]: CWI他们提出了一个专门针对向量搜索的存储格式[PDX](https://ir.cwi.nl/pub/35044/35044.pdf)，不过目前还是停留在学界研究
