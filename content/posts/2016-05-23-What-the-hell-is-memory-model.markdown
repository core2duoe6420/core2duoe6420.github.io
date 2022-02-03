---
title: "What the HELL is MEMORY MODEL"
date: 2016-05-23
slug: memory-model
---

这个标题看上去像是我能回答这个问题似的，事实上，我不能，我离回答这个问题差了十万八千里。这篇文章只是总结一下我目前所看到的资料以及自己理解的内容，完全不保证准确性。

<!--more-->

## 乱序执行

程序在执行时并不会按照源代码中所写的顺序执行，有好几种因素会导致实际执行顺序与我们所写的不同：

- 编译器进行了代码重排；
- CPU在执行时会进行指令重排（为了优化流水线、分支预测等等）；
- 缓存一致性协议延迟导致的CPU间数据不一致的问题，表现出来的症状与乱序执行很相似（关于这点，可以看[这篇文章](http://www.puppetmastertrading.com/images/hwViewForSwHackers.pdf)，介绍了一个简单的缓存系统）。

不管是优化也好还是缓存问题也罢，导致的现象是一致的，就是在多线程编程中，多个处理器同时对共享变量的访问会产生反直觉的结果，出现与源代码中执行顺序相反的现象。一个最经典的例子是，源码中A操作在B操作之前，现在我观察到了B已经发生，然而观察A的时候却发现A没有发生（interesting，有种看《量子物理史话》的感觉）。

这种情况还会发生在硬件编程中，其实质是存在多个处理器同时读写共享变量导致的。而这种不确定的情况是不允许发生的（一个实际的例子见[这篇博客](https://jfdube.wordpress.com/2012/03/08/understanding-memory-ordering/)）。解决问题的方法是内存屏障（Memory Barrier）。

## 内存屏障

内存屏障是CPU指令集中的一些特殊指令，在运行这条指令后，处理器会根据指令的类别同步内存和不同CPU间的缓存、对流水线分支预测做出修改（无责任猜想）等等，以使得指令的执行顺序符合程序员所要求的那样。不同架构的处理器的指令自然是不同的，因此，大致上将内存屏障分为以下4类：

- 读读屏障（`LoadLoad`）：屏障之前的读操作不能被乱序到屏障之后的读操作之后；
- 读写屏障（`LoadStore`）：屏障之前的读操作不能被乱序到屏障之后的写操作之后；
- 写写屏障（`StoreStore`）：屏障之前的写操作不能被乱序到屏障之后的写操作之后；
- 写读屏障（`StoreLoad`）：屏障之前的写操作不能被乱序到屏障之后的读操作之后；

不同架构的CPU使用的指令与上述四类屏障并不是一一对应的关系。例如，PowerPC上的`lwsync`指令同时具有`LoadLoad` `LoadStore` `StoreStore`三种屏障的功能。而我们常用的x86默认的对内存的存取操作都含有这三种屏障的功能（这种说法并不准确），类似x86这样的处理器架构也被称为强内存模型（Strong Memory Model），与之对应的，PowerPC、ARM就是弱内存模型（Weak Memory Model）。`StoreLoad`屏障尤为特殊，因为其代价高昂，原因大概是因为CPU会使用Store Buffer缓存写操作的结果，使用`StoreLoad`屏障要求Store Buffer的操作完成后才进行读操作，会使得读操作的延时大很多。

大多数现代处理器都会保证依赖读（Dependent Load）的顺序，也就是类似这样的操作：`q = p; x = *p;`，主流CPU中只有DEC Alpha不保证这种顺序，因此需要在这两条指令间加入一个依赖读屏障（Dependent Load Barrier）（注意有另一个CPU同时在改变`p`的值，详见[Linux Kernel文档](https://www.kernel.org/doc/Documentation/memory-barriers.txt)）。Alpha这样的处理器也被称为最松散（most relaxed）内存模型。

## 内存模型

关于内存模型（Memory Model）的定义，维基百科上的说法是：内存模型描述了线程间应该怎么通过共享内存来进行交互（这个译法取自[该文](http://www.cnblogs.com/catch/p/3803130.html)）。我个人的理解是程序语言的内存模型为程序员提供了一致的内存模型视图，而具体到不同的硬件平台时，就需要映射到不同CPU内存模型上（使用各个架构提供的内存屏障指令）。语言为了兼顾可移植性，其设计的内存模型必然更加通用，更有可能是最有硬件的最小公倍数。因此在x86上，使用`std::memory_order_relaxed`不会有实际的效果，但是为了使程序可移植，并且在不同的硬件平台上性能最大化，就要针对C++提供的内存模型进行编程。

#### Acquire和Release语义

关于Acquire和Release语义，网上也是众说纷纭，有将其分开定义的，也有将其合并定义的，相比之下，我还是更喜欢后者，这也是Herb Sutter在他关于内存模型的talk（[part1](https://channel9.msdn.com/Shows/Going+Deep/Cpp-and-Beyond-2012-Herb-Sutter-atomic-Weapons-1-of-2) [part2](https://channel9.msdn.com/Shows/Going+Deep/Cpp-and-Beyond-2012-Herb-Sutter-atomic-Weapons-2-of-2)）上给出的定义，如下：

> 如果一个线程A对一块内存`m`以Release的方式进行修改，那么在线程A中，所有在该release操作之前进行的内存操作，都在另一个线程B对内存`m`以Acquire的方式进行读取之后，变得可见。

另一个我比较喜欢的定义是使用前一节介绍的4种内存屏障的组合（见[该文](http://blog.forecode.com/2010/01/29/barriers-to-understanding-memory-barriers/)和[该文](http://preshing.com/20120913/acquire-and-release-semantics/)），属于一种映射关系：

> Acquire == `LoadLoad + LoadStore`
>
> Release == `LoadStore + StoreStore`

用文字表述的话，就是：

- Acquire语义仅可用于共享内存的读操作上。这样的读操作被称作`read-acquire`，Acquire语义阻止任何在Acquire操作之后的内存读写操作被乱序到Acquire操作之前；
- Release语义仅可用于共享内存的写操作上。这样的写操作被称作`write-release`，Release语义阻止任何在Release操作之前的内存读写操作被乱序到Release操作之后。

此外Acquire和Release语义还有相应的Fence操作。

Acquire和Release语义必须成对使用，不然不会有效果，这也是为什么把它们放在一起使用更合适的原因。在C++11标准中，`write-release`操作与`read-acquire`操作可以产生`Synchronize-With` 关系，从而使两个线程中在`write-release`操作之前的内存操作与`read-acquire`操作之后的内存操作产生明确的`Happen-Before`关系，使得本文开始所说的“明明B已经发生A却没有发生”的情况不会出现。

之前还提到语言提供的内存模型通常和硬件提供的内存模型不是一一对应的。Herb Sutter在他的talk中还提到最新的ARMv8提供了与`write-release`和`read-acquire`对应的读写指令`ldra`和`strl`，是第一个与软件内存模型相一致的硬件内存模型，理论上可以提高性能。

#### 顺序一致与SC-DRF

顺序一致（Sequential Consistency）是比Acquire-Release语义更严格的语义，它要求所有的内存操作所产生的效果都有一个全局的顺序，如果一个线程认为操作x发生在操作y之前，那么所有的线程都认为操作x发生在操作y之前。JAVA和C#中被`volatile`修饰的变量应该具有顺序一致语义。

我并不是很能理解顺序一致究竟是如何实现的，基本上在所有的体系结构上顺序一致都要求使用全屏障（Full Barrier），这里面究竟发生了什么事情，处理器如何同步缓存和内存的，只有深入研究各个体系结构才能知道了。

cppreference上有一个关于顺序一致的极其反人类的例子：

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

这个例子只有在使用顺序一致的情况下才能正常运行，使用其它模型（Acquire-Release语义）都会使assert触发。这个例子应该必须使用四个CPU才能有效果，并且我估计多半是因为CPU缓存不同步（而不是因为指令被重排了）导致的问题。虽然我现在可以接受这个例子，但仍然觉得它实在是太恐怖了。

Herb Sutter还提到了一个词，SC-DRF（Sequential Consistent - Data Race Free）。[该页](http://cs.stackexchange.com/questions/29043/why-is-a-program-with-only-atomics-in-sc-drf-but-not-in-hrf-direct)中的回答解释道：“SC-DRF将内存位置分为两类，一类是同步对象，一类是数据对象。对于大多数同步对象上的操作都保证是顺序一致的。如果程序员说一个线程先写入原子对象`a`，再写入源自对象`b`，那么所有的线程都将观察到同样顺序的写入操作。”

Herb Sutter的talk的后半段我并没有听懂多少（听力太差），关于这个确实不明白，先记一笔今后再看。

## 结束语

第一次看到Memory Model这个词是在看《C++ Concurrency in Action》这本书中，其中第五章介绍了相关内容，当时完全没有看懂。实际上在更早的时候看过的《CLR via C#》中应该也提到了Memory Order的相关问题，但也是觉得晦涩难懂。究其原因，我认为是因为的我的学习很大程度上基于对具体事物的理解，向来就对抽象的东西不感冒。Memory Order问题并不抽象，但一般只有写lock-free代码时才会碰到，在x86上就更少见了。或许我可以买个树莓派在ARM上试试看，不过那是以后的事了。这次看Memory Model相关的问题也只是因为现在在写的代码要用到`std::atomic<>`，觉得不理解Memory Model用起来不踏实，结果一看就是2天时间，代码都没写，看了之后还是云里雾里，并不踏实，只能说惭愧，智商不够用。这篇文章算是两天看下来的总结，暂时告一段落了。
