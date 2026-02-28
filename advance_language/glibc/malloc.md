# 1.2.1 malloc 机制

glibc 默认 malloc 实现是 **ptmalloc2**，本质是 dlmalloc + 多线程 arena 扩展。

## 整体架构

```
用户态malloc -> __libc_malloc -> arena_get() -> tcache(thread local) -> fastbin -> smallbin -> largebin -> unsorted bin -> top chunk -> sysmalloc() -> brk/mmap -> linux kernel
```

## 核心概念

### Arena

一个分配区域（arena）包含：

- 一个或多个 heap
- 该 arena 中管理的 chunks 的状态链表

**主 arena：**

- 初始化时最开始的 heap
- 单线程程序大多数只用主 arena

**多 arena 目的：**

- 减少多线程竞争
- 不同线程尽量分配不同 arena 的内存

**arena 与线程的关系：**

- 每个 arena 有 mutex 锁保护
- 线程竞争 arena 内部数据结构会加锁

**arena 最大数量默认：** [mallopt(3) - Linux man page](https://man7.org/linux/man-pages/man3/mallopt.3.html)

- 64-bit 系统：8 * ncpu
- 32-bit 系统不同配置

**Arena 存储：**

- fastbins
- unsorted bin
- smallbins
- largebins
- top chunk
- arena mutex
- 所属线程计数等

#### fastbins

Small chunks are stored in size-specific bins. Chunks added to a fast bin ("fastbin") are not combined with adjacent chunks - the logic is minimal to keep access fast (hence the name). Chunks in the fastbins may be moved to other bins as needed. Fastbin chunks are stored in an array of singly-linked lists, since they're all the same size and chunks in the middle of the list need never be accessed.


#### smallbins

The normal bins are divided into "small" bins, where each chunk is the same size, and "large" bins, where chunks are a range of sizes. When a chunk is added to these bins, they're first combined with adjacent chunks to "coalesce" them into larger chunks. Thus, these chunks are never adjacent to other such chunks (although they may be adjacent to fast or unsorted chunks, and of course in-use chunks). Small and large chunks are doubly-linked so that chunks may be removed from the middle (such as when they're combined with newly free'd chunks).

#### largebins

A chunk is "large" if its bin may contain more than one size. For small bins, you can pick the first chunk and just use it. For large bins, you have to find the "best" chunk, and possibly split it into two chunks (one the size you need, and one for the remainder).

#### unsorted bin

When chunks are free'd they're initially stored in a single bin. They're sorted later, in malloc, in order to give them one chance to be quickly re-used. This also means that the sorting logic only needs to exist at one point - everyone else just puts free'd chunks into this bin, and they'll get sorted later. The "unsorted" bin is simply the first of the regular bins.

### Heap

heap 是一块可增长的连续内存区域：

- 最初由 brk（sbrk）扩展
- 大块分配可能使用 mmap

每个 heap 属于一个 arena。

### Chunk

一个 chunk 代表着一块内存区域及其元数据：

- **chunk header** 包含：
  - size
  - prev_size
  - 双向链表指针（用于 free 状态）
- chunk 既可以是已分配给用户的内存
- 也可以是 free 状态并放入链表管理

核心数据结构为 `malloc_chunk`。

### tcache

glibc 2.26+ 引入的优化：

- 每个线程都有一个局部的 tcache
- 对于频繁 alloc/free 的小块请求：
  - 优先从 tcache 取
  - 不用上锁
  - 显著加快 small alloc/free

**tcache 维护：**

- 每个大小分类的单向链表
- 每个分类最多保存固定数量
- 配置由 mallopt 控制

## malloc 分配算法

先检查 **tcache**（如果启用） -> 如果申请大小大于某个阈值，则使用 **mmap**（优点：free 时直接释放给操作系统） -> **fastbins**（小块，不合并，单链表） -> **smallbins/largebins**（中等、较大，有序链表）-> **unsorted bins**（新 free 的块先放入这里，malloc 时会先检查 unsorted bins，有机会合并并重新分类） -> **top chunk**（这里拿新的内存，如果不够，则 sysmalloc 向系统申请）

## free 算法

尝试放回 **tcache**（如果启用） -> **fastbins/smallbins/largebins**（free 会合并邻近空闲块，即邻近都 free 则合并） -> **coalesce**（块合并。利用 chunk header 中的标记位，连续空闲的块合并成大块，减少碎片）

## ptmalloc 存在的问题

- arena 数量不可控 → 内存膨胀
- 锁竞争仍然存在
- 碎片率较高
- NUMA 友好性差

## 与其他 malloc 实现对比

- **glibc** —“历史演进型 allocator”
- **jemalloc** —“碎片控制型 allocator”
- **TCMalloc** — “吞吐优化型 allocator”

| 特性 | libc malloc | jemalloc | TCMalloc |
| --- | --- | --- | --- |
| 多线程优化 | arena+tcache | 更细粒度 | 更 aggressive |
| 内存碎片 | 中等 | 优秀 | 取决于场景 |
| 兼容性 | 标准 API | 标准 API | 标准 API |
| 安全防护 | 部分 | 部分 | 部分 |
| 历史 | dlmalloc 衍生 | 现代重构 | Google 定制 |
| 小对象 | bin+链表 | slab | thread list |
| 多线程 | arena+mutex | arena+tcache | thread cache |
| 碎片 | 中等 | 低 | 中等 |
| 复杂度 | 非常高 | 高 | 中 |

### jemalloc

TODO

### TCMalloc

TODO

## 现代 allocator 设计核心主题

### 内存分配算法基础

- First fit / Best fit / Next fit
- Segregated free lists
- Buddy system
- Bitmap / slab / slab+bitmap

| 算法 | 时间复杂度 | 内碎片 | 外碎片 | 并发友好 | 应用 |
| --- | --- | --- | --- | --- | --- |
| First Fit | O(n) | 低 | 高 | 差 | 早期 malloc |
| Best Fit | O(n) | 低 | 高 | 差 | 理论 |
| Next Fit | O(n) | 低 | 高 | 差 | 过渡 |
| Segregated | O(1) | 中 | 中 | 好 | glibc |
| Buddy | O(log n) | 高 | 低 | 好 | OS |
| Bitmap | O(n) | 低 | 低 | 中 | 嵌入式 |
| Slab | O(1) | 低 | 极低 | 极好 | jemalloc |

#### First fit / Best fit / Next fit

#### Segregated free lists

#### Buddy system

#### Bitmap / slab / slab+bitmap

现代 allocator 是如何组合这些算法的？
现代设计不是单一算法，而是组合：
例如 jemalloc：
小对象 → slab + bitmap
中对象 → segregated
大对象 → extent tree

TCMalloc：
小对象 → thread free list
中对象 → central free list
大对象 → page heap (buddy-like)

glibc：
fastbin → segregated
smallbin → double linked list
largebin → sorted list


### 多线程设计

- Arena / Heap 分离
- Thread Local Cache
- Lock contention
- Batch allocation

### 缓存友好 & NUMA

- Cache line 对齐
- False sharing
- NUMA locality

### 低碎片策略

- Slab
- Size class 粒度优化
- 分配算法对持续服务的影响

### 安全

- Double free mitigation
- Safe unlinking
- pointer mangling
- heap hardening

## 参考

- [glibc Malloc Internals](https://sourceware.org/glibc/wiki/MallocInternals)
- [The Memory Management Reference – Hans Boehm](https://www.memorymanagement.org/bib.html)
- [malloc Internals and the Linux Memory Allocator — by Michael Kerrisk]()
- [Understanding and Writing a Memory Allocator — Justin Meiners]()
- [Systems Performance: Enterprise and the Cloud — Brendan Gregg]()
- [Dynamic Storage Allocation: A Survey and Critical Review — Paul R. Wilson et al.](https://www.cs.hmc.edu/~oneill/gc-library/Wilson-Alloc-Survey-1995.pdf)
- [The Slab Allocator: An Object-Cashing Kernel Memory Allocator — Jeff Bonwick](https://people.eecs.berkeley.edu/~kubitron/courses/cs194-24-S14/hand-outs/bonwick_slab.pdf)
- [TCMalloc: Thread-Caching Malloc — Sanjay Ghemawat, Paul Men-Frederick](https://gperftools.github.io/gperftools/tcmalloc.html)
- [Scalable Memory Allocation Using Automatic Private Heaps — David Lea et al.]()
- [jemalloc: Memory Allocation for Threads](https://people.freebsd.org/~jasone/jemalloc/bsdcan2006/jemalloc.pdf)
