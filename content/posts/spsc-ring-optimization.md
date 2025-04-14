+++
title = "SPSC 环形队列优化"
date = "2024-12-26"

[taxonomies]
tags = ["entry"]
+++

# Cache

来源 <https://www.reddit.com/r/rust/comments/1h3bqv0/why_is_ringbuf_crate_so_fast/>，

> 5x sounds like about the speed up you get from having cached indices, where the reader caches the write head and the writer caches the read tail, and they only update those to the “real” atomic values of the other when they run out of items or space, respectively.

> No I don’t mean simply padding out the cache line, I mean there are two copies of the read index and two copies of the write index, one of each is the atomic that they write to, and the other is non-atomic which the opposite thread reads from

重点在于减少了 atomic 的使用。

因为在 SPSC 模式下，Writer 读取的 `head` 指针只是用来判断队列是否已满，因此这个指针完全可以被拿来缓存，这样在大量数据的情况下非常有用。Reader 同理，可以缓存 `tail` 指针来判断是否为空，仅当判断到队列为空时才会执行一次 atomic 来更新对应的指针。

另外还需要考虑 cache 的 false sharing 问题，所以最好每个 atomic pointer 都最好 padding 一下。

又学到了没啥用的知识，好耶😆！

In detail: [https://rigtorp.se/ringbuffer](https://rigtorp.se/ringbuffer/)

# 尝试理解

其实无锁这个东西（特指 Atomic）要想深入还是很复杂的，主要是要考虑到内存序的问题，这虽然也是个老生常谈的话题了。

其实总的来看，内存序（**只针对单个核心**）主要就是 load store 的四种排列组合，

* Load-Load，两个 Load 指令需要被同步
* Load-Store，前面的 Load 指令与后面的 Store 指令需要被同步
* Store-Store，两个 Store 指令需要被同步
* Store-Load，前面的 Store 指令与后面的 Load 指令需要被同步

这里的指令粒度都只是指令本身，跟操作的内存区域无关。例如 ARM64，只要你是按照字节对齐的方式（访问多少字节就需要多少字节对齐，例如访问一个 `u64` 变量就需要是 8 字节对齐的），那么都能保证指令的原子性（执行完指令后其他 CPU 看到的地址内容一定是最新的）。

<https://davekilian.com/acquire-release.html>

<https://en.wikipedia.org/wiki/Load-Hit-Store>

# 怎么选择

其实说了这么多原理，最重要的还是要决定选择哪种内存序，非常推荐 [Preshing](https://preshing.com/20120913/acquire-and-release-semantics/) 的这篇文章，通常来说选择一般就是三种，

* Relaxed，不带任何约束，只保证原子性
* Acquire
* Release
* Sequence，不是很常用，基本上只有在不确定的时候才选择

 ![From Preshing](/api/attachments.redirect?id=e042ec5b-a0b3-4c48-b777-33f38ccf3ec1)

这个图其实就能看的比较清晰了：

对于 Acquire，始终需要保证之前的 Load 操作完成，然后才能执行后续的 Load 或者 Store。

对于 Release，始终需要保证 Store 之前的 Load 或者 Store 操作完成，然后才能执行 Store。

主要是提供 locking 相关的功能的，例如 spinlock 这个操作，

```c
spin_lock(&lock);
global_vars_a *= 2;
spin_unlock(&lock);
```

比较典型的一个场景，因为真实中 `global_vars_a *= 2` 的这个操作会分为三步，一步先从内存中读取 `global_vars_a` 到寄存器中，然后在寄存器完成乘法或位运算，最后将寄存器中算好的值重新回写到内存里。

那么我们假设这个 spinlock 没有任何内存序保护，就有可能回出现 `global_vars_a` 先于 `&lock` 被读取！

```c
register a0 = READ_ONCE(global_vars_a);
spin_lock(&lock);
a0 *= 2;
```

这就是所谓的乱序发生了，这是会直接导致 data race 的，因为 spinlock 没能将变量的读取保护在临界区内（读取完锁的状态后）。

回写也是同理，如果没有内存序保护，就会发生可能已经 spin_unlock 了之后 `global_vars_a` 才被写回去的情况。

# Store-Load

这个我觉得其实是个有趣的问题，为什么如此常用的 acquire 和 release 语义居然不保证 Store-Load 内存序呢？

还是 [Preshing](https://preshing.com/20170612/can-reordering-of-release-acquire-operations-introduce-deadlock/) 指出了这一点，文中的一个例子，

```cpp
A.store(1, std::memory_order_release);
int b = B.load(std::memory_order_acquire);
```

因为 Store-Load 序的缺失，所以实际上 CPU 可以把他重排成（等价的语言描述）

```cpp
int b = B.load(std::memory_order_acquire);
A.store(1, std::memory_order_release);
```

这里的 A 和 B 两个 atomic 是可能被重排的。

其实如果是同一个变量，或者说变量之间存在指针等依赖关系时，如 ARM64 这种架构实际上是有 [data dependency](https://duetorun.com/blog/20231006/a64-memory-ordering/#secure_order) 这样的概念的，它保证了同一个变量下的 Store-Load 序。但是对于不同的变量来说，它并不能完全保证。

并且的并且，ARM64 实际上有 `stlr` 和 `ldar` 这两个指令，它们在 Acquire Release 的基础上也增加了对 Store-Load 序的一些保证，相当于部分的 SC（Acquire 和 Release 分别都多了这一个内存续保证）。

另外作者指出了 C++ 标准中的一些说明，

> *An implementation should ensure that the last value (in modification order) assigned by an atomic or synchronization operation will become visible to all other threads in a finite period of time.*

换言之就是只要我 Store 指令发出后，那么架构必须要保证在有限的时间内其他 Load 指令最终可以看到这个结果，这样就避免了很多可能存在的问题。

```cpp
std::atomic<int> A = 0;
std::atomic<int> B = 0;

void thread1() {
    A.store(1, std::memory_order_release);

    while (B.load(std::memory_order_acquire) == 0) {
    }
}

void thread2() {
    while (A.load(std::memory_order_acquire) == 0) {
    }

    B.store(1, std::memory_order_release);
}
```

这个例子里，假设没有标准约束的话，它是可能会被重排成

```cpp
void thread1() {
    while (B.load(std::memory_order_acquire) == 0) {
    }

    A.store(1, std::memory_order_release);
}
```

这样就会陷入死锁的问题，但是有了上面标准后，既然目前编译器不会重排这个指令，那么 `A.store` 指令一定是先于 `B.load` 指令就发出了的，这里也就必须保证无论 `thread2` 自旋多久，它总能因为 `A.store` 执行成功而执行后面的 `B.store` ，并且这个时间需要是“finite”的。

这一条也其实就限制了处理器，即使你乱序执行，但是该执行的还是得执行。

[More about ARM64](https://stackoverflow.com/a/67397890)：

> Yup, stlr is store-release on its own, and ldar can't pass an earlier stlr (i.e. no StoreLoad reordering) - that interaction between them satisfies that part of the seq_cst requirements which acq / rel doesn't have. ([ARMv8.3 ldapr](https://community.arm.com/groups/processors/blog/2016/10/27/armv8-a-architecture-2016-additions) is like ldar without that interaction, being only a plain acquire load, allowing more efficient acq_rel.)

也就是说 `ldar` 和 `stlr` 之间是被排序了的，保证了 Store-Release 顺序。

## LDAPR

```cpp
AccessDescriptor CreateAccDescLDAcqPC(boolean tagchecked)
  AccessDescriptor accdesc = NewAccDesc(AccessType_GPR);
  accdesc.acqpc         = TRUE;
  accdesc.read          = TRUE;
  accdesc.pan           = TRUE;
  accdesc.tagchecked    = tagchecked;
  accdesc.transactional = IsFeatureImplemented(FEAT_TME) && TSTATE.depth > 0;

  return accdesc;
```

## LDAR

```cpp
AccessDescriptor CreateAccDescAcqRel(MemOp memop, boolean tagchecked)
  AccessDescriptor accdesc = NewAccDesc(AccessType_GPR);
  accdesc.acqsc         = memop == MemOp_LOAD;
  accdesc.relsc         = memop == MemOp_STORE;
  accdesc.read          = memop == MemOp_LOAD;
  accdesc.write         = memop == MemOp_STORE;
  accdesc.pan           = TRUE;
  accdesc.tagchecked    = tagchecked;
  accdesc.transactional = IsFeatureImplemented(FEAT_TME) && TSTATE.depth > 0;
  
  return accdesc;
```

可以看到 `ldar` 果然已经叫 `acqsc` 和 `relsc` 了，也就是 SC Acqure/Release 了吧（。

还可以继续看一下 `AccessDescriptor` 的相关定义，

```cpp
boolean acqsc,          // Acquire with Sequential Consistency
boolean acqpc,          // FEAT_LRCPC: Acquire with Processor Consistency
boolean relsc,          // Release with Sequential Consistency
boolean limitedordered, // FEAT_LOR: Acquire/Release with limited ordering
```

CPU，真神奇。

# Power-of-2

<https://max0x7ba.github.io/atomic_queue/#ring-buffer-capacity>

> The writer and reader indexes get mapped into the ring-buffer array index using remainder binary operator % SIZE. Remainder binary operator % normally generates a division CPU instruction which isn’t cheap, but using a power-of-2 size turns that remainder operator into one cheap binary and CPU instruction and that is as fast as it gets.

以及

> The element index within the cache line gets swapped with the cache line index, so that consecutive queue elements reside in different cache lines. This massively reduces cache line contention between multiple producers and multiple consumers. Instead of N producers together with M consumers competing on subsequent elements in the same ring-buffer cache line in the worst case, it is only one producer competing with one consumer (pedantically, when the number of CPUs is not greater than the number of elements that can fit in one cache line). This optimisation scales better with the number of producers and consumers, and element size. With low number of producers and consumers (up to about 2 of each in these benchmarks) disabling this optimisation may yield better throughput (but higher variance across runs).

当然这个只是比较“performance”的原因。
