---
title: "Kotlin Coroutine笔记"
date: 2022-04-06
slug: kotlin-coroutine-note
tags:
- kotlin
---

这两天在死磕Kotlin协程的原理。看了很多资料，从一开始感觉混乱没有头绪到现在稍微有了点感觉，本文记录了我目前对Kotlin协程的理解。

<!--more-->

先上各种参考资料：

1. [Kotlin Coroutines](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)：这是Kotlin协程的设计提案，强烈建议阅读
1. [破解Kotlin协程 系列](https://www.bennyhuo.com/2019/04/01/basic-coroutines/)
1. [深入理解Kotlin协程 系列](https://www.sukaidev.top/2021/02/01/595755ca/)
1. [Why Synchronized suspend doesn't work in Kotlin](https://linuxtut.com/why-does-synchronized-suspend-not-work-in-kotlin-45b45/)
1. [Kotlin笔记之协程工作原理](https://ljd1996.github.io/2021/05/19/Kotlin%E7%AC%94%E8%AE%B0%E4%B9%8B%E5%8D%8F%E7%A8%8B%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/)
1. [Kotlin协程分析（二）——suspendCoroutineUninterceptedOrReturn](https://codeantenna.com/a/s6eHPJ3nvs)

## 概念

这里先理清一些术语和概念，内容基本上取自Kotlin协程的设计提案：

- `coroutine 协程`：协程是一个可挂起的程序的实例。协程与线程一样可以承载运行一段程序，并且具有类似的生命周期：创建，运行，并且具有一些交互语义，比如`.join()`。协程可以在一个线程上*挂起*（suspend），然后在另一个线程上*恢复*（resume）执行。此外，协程可以像`future`或是`promise`那样返回一个结果（可以是一个值或者一个异常）
- `suspending function 挂起函数`：一个标注了`suspend`描述符的函数。挂起函数可以通过调用另一个挂起函数在执行过程中挂起，然而并不阻塞当前的线程。挂起函数只能被另一个挂起函数或者挂起匿名函数调用，普通的代码不能调用挂起函数。标准库提供了一些基本挂起函数用于定义其它所有的挂起函数
- `suspending lambda 挂起匿名函数`：一个只能运行在协程中的代码段。挂起匿名函数看上去与普通匿名函数（lambda表达式）一致，但它的类型被标注了`suspend`。例如`launch`之后的代码块就是挂起匿名函数
- `suspending function type 挂起函数类型`：是挂起函数和挂起匿名函数的函数类型
- `couroutine builder 协程构造器`：一个接受挂起匿名函数为参数，创建一个协程，并且允许返回执行结果的函数。比如`lanuch {}`，`sequence {}`，`async {}`都是协程构造器
- `suspension point 挂起点`：是在协程运行中可以挂起的位置。从语法上讲，挂起点就是挂起函数调用的位置，但是实际的挂起动作发生在挂起函数调用定义在标准库中的基本挂起函数的时候
- `continuation`：描述了挂起的协程在挂起点的状态。从概念上说，它代表了在挂起点之后要执行的代码

## continuation

continuation的概念非常重要，基本上理解了continuation就能理解Kotlin协程是怎么工作的，整个协程的实现也基本上是围绕continuation的。简单地来说，continuation就是回调函数，而协程就是一连串的continuation链接在一起组成的。

```kotlin
suspend fun foo() { 
    delay(1000)
    println("after delay")
    ...
}
```

对这个挂起函数来说，`delay()`是个挂起点，`delay`（第2行）之后的程序就是这个挂起点的continuation。如果翻译成Javascript里那种回调形式，就类似于（伪代码，方便理解，不能执行）：

```js
function foo() {
    delay(
        1000, 
        function() {    <-- 这个函数可以理解为continuation
            println("after delay")
            ...
        }
    )
}
```

那么来看一下continuation的定义：

{{< codeblock Continuation.kt kotlin "https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/Continuation.kt" >}}
interface Continuation<in T> {
    val context: CoroutineContext
    fun resumeWith(result: Result<T>)
}
{{< /codeblock >}}

这里的`resumeWith`就可以理解为是一个回调函数。

每个挂起函数在编译时都会经过一个叫Continuation-Passing-Style (CPS)的变换。经过变换，挂起函数会多一个参数`continuation`，这个`continuation`就是本挂起函数执行完毕后，要恢复执行的“回调函数”。

举个例子

```kotlin
suspend fun foo(a: Int): Int
```

在编译后会变成：

```kotlin
fun foo(a: Int, continuation: Continuation<Int>): Any?
```

注意到`foo()`的返回值被编译器改写成了`Any?`，这是因为除了原先的返回值类型外，所有的挂起函数都可能返回`COROUTINE_SUSPENDED`这个特殊值。当返回值是`COROUTINE_SUSPENDED`时，整个协程的调用栈会一路返回，从而让出线程。此时通常会有一个调度器负责调度另一个协程开始执行，但注意调度器并不是必要的，比如在生成器（Generator）中，就没有调度器，线程的使用权会在生成器的协程与生成器的使用者之间来回切换。

那么问题来了，谁是这一切的罪魁祸首？也就是，谁第一个返回`COROUTINE_SUSPENDED`导致整个协程被挂起的？这就要提到前面概念中的“基本挂起函数”，比如下面这个函数就是一个基本挂起函数：

{{< codeblock Continuation.kt kotlin "https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/Continuation.kt" >}}
public suspend inline fun <T> suspendCoroutine(crossinline block: (Continuation<T>) -> Unit): T {
    contract { callsInPlace(block, InvocationKind.EXACTLY_ONCE) }
    return suspendCoroutineUninterceptedOrReturn { c: Continuation<T> ->
        val safe = SafeContinuation(c.intercepted())
        block(safe)
        safe.getOrThrow()
    }
}
{{< /codeblock >}}

调用这个函数后，如果`block`没有同步调用`continuation`的`resumeWith`方法，就会使协程挂起。这个逻辑是在`SafeContinuation`这个`Continuation`的实现类里完成的，它同样保证了如果`resumeWith`方法没有被调用，会返回`COROUTINE_SUSPENDED`。

这个函数在把回调函数改写为挂起函数时经常用到，比如：

```kotlin
suspend fun foo(): Int = suspendCoroutine { continuation ->
    thread {
        Thread.sleep(1000)   // 这里相当于异步调用做了些操作，得到结果后用continuation.resume返回
        continuation.resumeWith(Result.success(42))
    }
}
```

注意到`suspendCoroutineUninterceptedOrReturn`这个函数没有？这是一个比`suspendCoroutine`还要基本的函数，这个函数的实现在标准库里是找不到的，它在Kotlin内部实现，据说直接操作字节码。它的作用其实是为了获得当前调用位置的`continuation`对象（就是代码里的那个`c`），否则这个`continuation`本身是由编译器生成的（下面会说），我们在Kotlin代码里是没法通过常规手段获取到这个对象的。

当`foo()`内的异步操作执行完毕后，就会去调用`continuation.resumeWith`，并将自己的返回值（或异常）作为输入传入，然后之前调用`foo()`的挂起函数就会恢复执行（可以理解为调用`foo()`的那个函数把`foo`之后的代码包装成了一个continuation，作为回调函数传给`foo`）。

编译器通过在所有的挂起函数中传递调用方的continuation实际上形成了一个链表，这个链表模拟了调用堆栈，依次执行`resumeWith`就能完成整个协程的执行。

## `Continuation`的实现

`Continuation`既然是个接口，就一定有实现，从功能上来说大体上有三类（我自己总结的，但是我研究的不深，不一定对）：

1. 挂起函数和挂起匿名函数会被编译器自动编译成`ContinuationImpl`的子类，继承链是这样的：`ContinuationImpl` -> `BaseContinuationImpl` -> `Continuation`。这类Continuation的`resumeWith`负责实际执行我们编写的代码，是真正干活的（实际上我们的代码会被放入`invokeSuspend()`函数中，`BaseContinuationImpl`的`resumeWith`会去调用`invokeSuspend()`，算是模板方法的应用）
2. 协程生成器，如`launch`，`async`之类，它们返回的`Job`以及`Deferred`实例分别是`StandaloneCoroutine`（或`LazyStandaloneCoroutine`），`DeferredCoroutine`（或`LazyDeferredCoroutine`），这些类都继承自`AbstractCoroutine`，`AbstractCoroutine`又实现了`Continuation`，所以这些也都是`Continuation`的实现。不过它们的主要职责是用来管理协程的生命周期，协程原语的实现，父子协程的关系，它们的`resumeWith`也是为了这些目的服务的，通常用来实现状态的变更（例如协程执行完毕，状态变为`Completed`。它们基本上都是协程continuation链上的最后一环
3. 像`Continuation`这样的单个函数的接口可玩性非常多，代理模式，装饰器模式随便用。第三类实现主要是围绕`resumeWith`进行展开，给`Continuation`添加功能，例如前面提到的`SafeContinuation`就属于这类。协程的很多重量级功能也是通过这类实现的，例如`DispatchedContinuation`就是与调度器配合实现协程在不同线程上执行的，它主要就是在`resumeWith`的实现中把代理的continuation放到了别的线程上执行；还例如`CancellableContinuation`在`resumeWith`中检查协程是否被取消，这也是为什么Kotlin协程能在挂起点取消的原因。这类实现中通常会把它们代理的continuation命名为`delegate`，而前两类则会把continuation命名为`completion`，因为前者是代理，而后者是协程continuation链上的一部分

这里简单说下第一类的实现（因为最简单）。编译器会把挂起函数和挂起匿名函数编译成一个状态机，这个状态机里有一个`label`，以及函数中所有的局部变量，同时编译器会将函数体根据挂起点（调用挂起函数的地方）进行切割，按序号排列。当函数执行到一个挂起点时，会更新状态机里的`label`为下一个序号，同时保存所有的局部变量，然后挂起。等到恢复执行时，在`invokeSuspend()`（会在`resumeWith`中被调用）里会再次调用自身，从状态机里恢复局部变量以及`label`，然后直接跳到`label`所属的位置执行。状态机的具体实现可以参考资料#4，写的非常清晰明了。

## 拦截器和调度

协程上下文（Coroutine Context）是直接定义于`Continuation`接口，属于最底层的机制。可以简单地把它理解为具有键值对属性的Map。实际的实现则采用了类型安全的Key，Immutable的设计（类似于`String`那样值不可变）。通常一个Coroutine Scope下的所有协程会共享上下文。协程上下文并不特殊，我们完全可以自己创建一个上下文的键值对，然后在业务代码中使用，这就类似于`ThreadLocal`之于Thread，`Attribute`之于Servlet Request。

当然有一些上下文的Key是特殊的。拦截器（Interceptor）是第一类`Continuation`直接提供的机制之一，拦截器是基于协程上下文实现的，它的Key是`ContinuationInterceptor`。`ContinuationImpl`只提供了一个`intercepted()`方法，用于调用当前上下文中保存的拦截器，让拦截器有机会通过调用拦截器接口`fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>`包装自己，返回一个被代理的`Continuation`对象。几乎所有的基本挂起函数（只要名字里不含`Unintercepted`）内都会帮我们调用`intercepted()`方法确保机制的完整性。


协程调度器（Coroutine Dispatcher）则是基于拦截器实现的，`CoroutineDispatcher`实现了`ContinuationInterceptor`接口，它的`interceptContinuation()`实现则是返回一个`DispatchedContinuation`实例（我们上面说的第三类），所以实际的调度会在其`resumeWith`内发生。我们常用的`Dispatchers.IO`，`Dispatchers.Default`都是`CoroutineDispatcher`的子类的实例。

有一个问题是，既然调度器是基于拦截器实现的，而拦截器又是基于上下文实现的，那对于第一类`Continuation`实现来说，它们刚创建的时候是完全没有上下文的，是谁把上下文赋给它们的呢？答案在`ContinuationImpl`的构造函数里：

{{< codeblock ContinuationImpl.kt kotlin "https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/jvm/src/kotlin/coroutines/jvm/internal/ContinuationImpl.kt" >}}
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {
    constructor(completion: Continuation<Any?>?) : this(completion, completion?.context)
{{< /codeblock >}}

`ContinuationImpl`会从传入的`completion`里复制上下文对象。而每次挂起函数和挂起匿名函数执行前都会调用到`ContinuationImpl`的构造函数，因此上下文就被传递到了整个协程的continuation链上。

那么第一个`completion`的上下文是哪来的？答案就是上述的第二类`Continuation`实现，它们在构造各自的实例时，都会有一套默认的上下文，与协程构造器传入的上下文参数合并，然后作为`completion`传给协程构造器的`block`参数所编译出的`ContinuationImpl`。

## 结束语

本文的内容其实不算深入，没有牵扯到生命周期，协程父子关系，协程取消之类的（第二类第三类实现）的话题，因为我没有深入了解。这跟我的兴趣点有关，我对最基础的概念和机制更感兴趣，对于我自身知识体系无法解释的问题会格外关注。而掌握了基础之后上层的实现会比较自由，高级话题等以后实际运用遇到了问题再来研究。老实说我对各个语言的协程实现都有所了解，包括C#，Javascript，Python，Rust，但是也都不够深入，常常纠结于执行顺序和调用栈的问题，这次研究Kotlin的协程实现也算是补上一块心病，以后能更全面的理解协程。