---
title: "What the HELL is MEMORY MODEL"
date: 2016-05-23
slug: memory-model
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

This title looks as if I were capable of answering this question. In fact, I’m not; I’m miles away from being able to answer it. This article is only a summary of the material I’ve read so far and my own understanding of it, with absolutely no guarantee of accuracy.

<!--more-->

## Out-of-order execution

When a program runs, it does not execute strictly in the order written in the source code. Several factors can cause the actual execution order to differ from what we wrote:

- The compiler reorders code;
- The CPU reorders instructions when executing (to optimize the pipeline, branch prediction, etc.);
- Delays in cache coherence protocols lead to data inconsistencies between CPUs, whose symptoms look very similar to out-of-order execution (for this, see [this article](http://www.puppetmastertrading.com/images/hwViewForSwHackers.pdf), which introduces a simple cache system).

Whether due to optimizations or cache issues, the resulting phenomenon is the same: in multithreaded programming, multiple processors simultaneously accessing shared variables can lead to counterintuitive results, producing effects that appear to contradict the execution order in the source code. The classic example is that operation A comes before operation B in the source code; now I observe that B has already happened, yet when I look at A I discover that A has not happened (interesting, it feels a bit like reading “In Search of Schrödinger’s Cat”).

This can also occur in hardware programming, and the essence is that multiple processors are reading and writing shared variables at the same time. Such nondeterministic behavior is not allowed (for a concrete example, see [this blog post](https://jfdube.wordpress.com/2012/03/08/understanding-memory-ordering/)). The way to address the problem is the Memory Barrier.

## Memory barriers

Memory barriers are special instructions in the CPU instruction set. After executing such an instruction, the processor will, depending on the barrier type, synchronize memory and caches across CPUs, adjust pipeline branch prediction, and so on (wild guess, no guarantees), to ensure that instructions are executed in the order required by the programmer. Since processors on different architectures provide different instructions, memory barriers are roughly divided into the following four categories:

- Load-load barrier (`LoadLoad`): load operations before the barrier cannot be reordered to after load operations that come after the barrier;
- Load-store barrier (`LoadStore`): load operations before the barrier cannot be reordered to after store operations that come after the barrier;
- Store-store barrier (`StoreStore`): store operations before the barrier cannot be reordered to after store operations that come after the barrier;
- Store-load barrier (`StoreLoad`): store operations before the barrier cannot be reordered to after load operations that come after the barrier;

The instructions used by CPUs on different architectures do not necessarily have a one-to-one correspondence with the four types above. For example, the `lwsync` instruction on PowerPC provides the combined effects of `LoadLoad`, `LoadStore`, and `StoreStore` barriers. On the x86 processors we commonly use, normal memory accesses implicitly provide the effects of these three barrier types (this statement is not strictly accurate). Processor architectures like x86 are also called Strong Memory Models; by contrast, PowerPC and ARM are Weak Memory Models. The `StoreLoad` barrier is particularly special because it is very costly, likely because the CPU uses a Store Buffer to cache the results of write operations, and `StoreLoad` requires that operations in the Store Buffer be completed before subsequent loads can proceed, significantly increasing load latency.

Most modern processors guarantee the ordering of dependent loads, i.e., operations like `q = p; x = *p;`. Among mainstream CPUs, only DEC Alpha does not guarantee this ordering and therefore requires a Dependent Load Barrier between those two instructions (note that another CPU is simultaneously modifying the value of `p`; see the [Linux Kernel documentation](https://www.kernel.org/doc/Documentation/memory-barriers.txt) for details). Processors like Alpha are also called most relaxed memory models.

## Memory model

As for the definition of a memory model, Wikipedia says: a memory model describes how threads should interact through shared memory (this translation is taken from [this article](http://www.cnblogs.com/catch/p/3803130.html)). My own understanding is that a programming language’s memory model provides programmers with a unified view of memory, and when mapping to specific hardware platforms, it must then be translated into the memory models of different CPUs (using the barrier instructions each architecture provides). To maintain portability, a language’s memory model design must necessarily be more general, more likely being something like the greatest lower bound of various hardware models. Therefore, on x86, using `std::memory_order_relaxed` effectively has no impact, but in order to make programs portable and maximize performance on different hardware platforms, programmers must write code against the memory model provided by C++.

#### Acquire and Release semantics

There are many different explanations of Acquire and Release semantics online. Some define them separately; others define them together. I prefer the latter approach, which is also the one Herb Sutter uses in his memory model talks ([part1](https://channel9.msdn.com/Shows/Going+Deep/Cpp-and-Beyond-2012-Herb-Sutter-atomic-Weapons-1-of-2) [part2](https://channel9.msdn.com/Shows/Going+Deep/Cpp-and-Beyond-2012-Herb-Sutter-atomic-Weapons-2-of-2)), defined as follows:

> If thread A modifies some memory `m` with a release operation, then in thread A, all memory operations that occur before that release operation become visible to thread B only after B performs an acquire read of `m`.

Another definition I like uses combinations of the four memory barriers introduced in the previous section (see [this article](http://blog.forecode.com/2010/01/29/barriers-to-understanding-memory-barriers/) and [this article](http://preshing.com/20120913/acquire-and-release-semantics/)), essentially a mapping:

> Acquire == `LoadLoad + LoadStore`
>
> Release == `LoadStore + StoreStore`

In words:

- Acquire semantics can only be applied to reads of shared memory. Such reads are called `read-acquire`. Acquire semantics prevent any memory read/write operations after the acquire from being reordered before the acquire.
- Release semantics can only be applied to writes to shared memory. Such writes are called `write-release`. Release semantics prevent any memory read/write operations before the release from being reordered after the release.

There are also fence operations with Acquire and Release semantics.

Acquire and Release must be used in pairs; otherwise they have no effect. This is why it makes sense to discuss them together. In C++11, a `write-release` operation and a `read-acquire` operation can form a `synchronize-with` relationship, which in turn establishes a well-defined `happen-before` relationship between memory operations before the `write-release` in one thread and memory operations after the `read-acquire` in another thread. This prevents the phenomenon described at the beginning of this article, where “B has obviously happened but A has not.”

Earlier I also mentioned that language-level memory models usually do not have a one-to-one correspondence with hardware-level memory models. In his talk, Herb Sutter mentions that the latest ARMv8 provides load/store instructions `ldra` and `strl`, corresponding to `read-acquire` and `write-release`, making it the first hardware memory model consistent with a software memory model in this way, theoretically improving performance.

#### Sequential consistency and SC-DRF

Sequential Consistency is a stricter semantics than Acquire-Release. It requires that the effects of all memory operations appear in some global order: if one thread thinks operation x happens before operation y, then all threads must agree that x happens before y. Variables marked `volatile` in Java and C# should have sequentially consistent semantics.

I don’t really understand how sequential consistency is actually implemented. On almost all architectures, sequential consistency requires using a full barrier. What exactly happens there, how the processor synchronizes caches and memory—one can only know by carefully studying the details of each architecture.

cppreference gives an extremely mind-bending example of sequential consistency:

```cpp
#include <thread>
#include <atomic>
#include <cassert>
 
std::atomic<bool> x = {false};
std::atomic<bool> y = {false};
std::atomic<int> z = {0};
 
void write_x() {
    x.store(true, std::memory_order_seq_cst);
}
void write_y() {
    y.store(true, std::memory_order_seq_cst);
}
void read_x_then_y() {
    while (!x.load(std::memory_order_seq_cst))
        ;
    if (y.load(std::memory_order_seq_cst)) {
        ++z;
    }
}
void read_y_then_x() {
    while (!y.load(std::memory_order_seq_cst))
        ;
    if (x.load(std::memory_order_seq_cst)) {
        ++z;
    }
}
 
int main() {
    std::thread a(write_x);
    std::thread b(write_y);
    std::thread c(read_x_then_y);
    std::thread d(read_y_then_x);
    a.join(); b.join(); c.join(); d.join();
    assert(z.load() != 0);  // will never happen
}
```

This example works correctly only when using sequential consistency; using other models (Acquire-Release semantics) can cause the assert to fire. This example likely requires four CPUs to actually manifest the effect, and my guess is that it’s mostly caused by unsynchronized CPU caches (rather than instruction reordering). Although I can now accept this example, I still feel it’s downright terrifying.

Herb Sutter also mentioned the term SC-DRF (Sequential Consistent – Data Race Free). The answer on [this page](http://cs.stackexchange.com/questions/29043/why-is-a-program-with-only-atomics-in-sc-drf-but-not-in-hrf-direct) explains: “SC-DRF divides memory locations into two kinds: synchronization objects and data objects. Operations on most synchronization objects are guaranteed to be sequentially consistent. If a programmer says that a thread first writes to the atomic object `a`, and then writes to the data object `b`, then all threads will observe the same order of writes.”

I didn’t understand much of the latter half of Herb Sutter’s talk (my listening skills are too poor). I genuinely don’t understand this part yet, so I’m just jotting it down and will revisit it later.

## Conclusion

I first saw the term Memory Model when reading the book “C++ Concurrency in Action,” whose fifth chapter introduces related content; at the time I didn’t understand any of it. In fact, the earlier book “CLR via C#” that I had read probably also mentioned Memory Order related issues, but I found it equally obscure and difficult. The root cause, I think, is that my learning is largely based on understanding concrete things; I’ve never been keen on abstractions. Memory Order problems aren’t actually abstract, but generally only appear when writing lock-free code, and on x86 they’re even less common. Maybe I can buy a Raspberry Pi and try it on ARM, but that’s for the future. This time I looked into Memory Model issues only because the code I’m writing now needs `std::atomic<>`, and I feel uneasy using it without understanding the Memory Model. The result is that I spent two days reading with no coding done, and after reading I still feel confused and uneasy. I can only say I’m ashamed—my IQ seems insufficient. This article is a summary of what I’ve read over the past two days; I’ll leave it here for now.