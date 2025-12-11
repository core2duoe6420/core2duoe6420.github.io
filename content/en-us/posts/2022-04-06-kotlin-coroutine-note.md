---
title: "Kotlin Coroutine Notes"
date: 2022-04-06
slug: kotlin-coroutine-note
tags:
- kotlin
---

{{<alert warning>}}
This article is translated from Chinese to English by ChatGPT. There might be errors.
{{</alert>}}

In the past two days I’ve been digging into how Kotlin coroutines work under the hood. I’ve read a lot of materials, and went from feeling completely lost to now having some rough intuition. This post records my current understanding of Kotlin coroutines.

<!--more-->

First, here are the references:

1. [Kotlin Coroutines](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md): this is the design proposal of Kotlin coroutines, highly recommended
1. [破解Kotlin协程 系列](https://www.bennyhuo.com/2019/04/01/basic-coroutines/)
1. [深入理解Kotlin协程 系列](https://www.sukaidev.top/2021/02/01/595755ca/)
1. [Why Synchronized suspend doesn't work in Kotlin](https://linuxtut.com/why-does-synchronized-suspend-not-work-in-kotlin-45b45/)
1. [Kotlin笔记之协程工作原理](https://ljd1996.github.io/2021/05/19/Kotlin%E7%AC%94%E8%AE%B0%E4%B9%8B%E5%8D%8F%E7%A8%8B%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86/)
1. [Kotlin协程分析（二）——suspendCoroutineUninterceptedOrReturn](https://codeantenna.com/a/s6eHPJ3nvs)

## 概念

Let’s first clarify some terms and concepts. The content is basically taken from the Kotlin coroutines design proposal:

- `coroutine`: A coroutine is an instance of a program that can be suspended. Like a thread, a coroutine can carry and execute a piece of code and has a similar lifecycle: creation, running, and some interaction semantics such as `.join()`. A coroutine can *suspend* on one thread and then *resume* execution on another thread. In addition, a coroutine can return a result like a `future` or `promise` (either a value or an exception).
- `suspending function`: A function marked with the `suspend` modifier. A suspending function can suspend its execution by calling another suspending function, without blocking the current thread. A suspending function can only be called by another suspending function or a suspending lambda; normal code cannot call a suspending function. The standard library provides some primitive suspending functions used to define all other suspending functions.
- `suspending lambda`: A block of code that can only run inside a coroutine. A suspending lambda looks the same as a normal lambda expression, but its type is marked with `suspend`. For example, the block after `launch` is a suspending lambda.
- `suspending function type`: The function types of suspending functions and suspending lambdas.
- `couroutine builder`: A function that takes a suspending lambda as a parameter, creates a coroutine, and optionally returns its result. For example, `launch {}`, `sequence {}`, and `async {}` are all coroutine builders.
- `suspension point`: A point at which a running coroutine can be suspended. Syntactically, a suspension point is where a suspending function is called, but the actual suspension happens when that suspending function calls one of the primitive suspending functions defined in the standard library.
- `continuation`: Describes the state of a suspended coroutine at a suspension point. Conceptually, it represents the code to be executed after the suspension point.

## continuation

The concept of continuation is extremely important. Once you understand continuation, you can basically understand how Kotlin coroutines work. The whole implementation of coroutines is centered around continuation. Simply put, a continuation is a callback function, and a coroutine is a series of continuations chained together.

```kotlin
suspend fun foo() { 
    delay(1000)
    println("after delay")
    ...
}
```

For this suspending function, `delay()` is a suspension point, and the code after `delay` (line 2) is the continuation of this suspension point. If we translate it into the callback style in JavaScript, it would look like this (pseudo-code, for understanding only, not executable):

```js
function foo() {
    delay(
        1000, 
        function() {    <-- this function can be understood as the continuation
            println("after delay")
            ...
        }
    )
}
```

Now let’s look at the definition of continuation:

{{< codeblock Continuation.kt kotlin "https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/Continuation.kt" >}}
interface Continuation<in T> {
    val context: CoroutineContext
    fun resumeWith(result: Result<T>)
}
{{< /codeblock >}}

Here, `resumeWith` can be thought of as a callback function.

Every suspending function goes through a transformation called Continuation-Passing-Style (CPS) at compile time. After this transformation, the suspending function gets an extra parameter `continuation`, which is the “callback function” that resumes execution after this suspending function completes.

For example:

```kotlin
suspend fun foo(a: Int): Int
```

After compilation, it becomes:

```kotlin
fun foo(a: Int, continuation: Continuation<Int>): Any?
```

Note that the return type of `foo()` is rewritten by the compiler to `Any?`. This is because, in addition to the original return type, all suspending functions may return a special value `COROUTINE_SUSPENDED`. When the return value is `COROUTINE_SUSPENDED`, the entire coroutine call stack will unwind, yielding the thread. At this point, there is usually a dispatcher that schedules another coroutine to run. But note that a dispatcher is not strictly necessary. For example, in a generator, there is no dispatcher; the ownership of the thread switches back and forth between the generator’s coroutine and the generator’s consumer.

So the question is: who is the culprit that starts all this? In other words, who is the first to return `COROUTINE_SUSPENDED`, causing the whole coroutine to suspend? This brings us to the “primitive suspending functions” mentioned earlier. For example, the following function is a primitive suspending function:

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

After calling this function, if `block` does not synchronously call the `resumeWith` method of the `continuation`, the coroutine will be suspended. This behavior is implemented in `SafeContinuation`, an implementation of `Continuation`. It also guarantees that if `resumeWith` is not called, it will return `COROUTINE_SUSPENDED`.

This function is often used when converting callback-style APIs into suspending functions, for example:

```kotlin
suspend fun foo(): Int = suspendCoroutine { continuation ->
    thread {
        Thread.sleep(1000)   // this simulates an async call; after some work, we return the result via continuation.resume
        continuation.resumeWith(Result.success(42))
    }
}
```

Did you notice the function `suspendCoroutineUninterceptedOrReturn`? This is an even more primitive function than `suspendCoroutine`. Its implementation cannot be found in the standard library; it is implemented inside Kotlin itself and is said to manipulate bytecode directly. Its purpose is to obtain the `continuation` object at the current call site (the `c` in the code). Otherwise, this `continuation` is generated by the compiler (as we’ll discuss below), and we cannot obtain this object from Kotlin code by normal means.

When the async operation inside `foo()` completes, it will call `continuation.resumeWith`, passing its return value (or exception) as input. Then the suspending function that previously called `foo()` will resume execution (you can think of it as: the function calling `foo()` wraps the code after `foo` into a continuation and passes it as a callback to `foo`).

By passing the caller’s continuation through all suspending functions, the compiler effectively creates a linked list. This linked list simulates the call stack, and by invoking `resumeWith` one by one, the entire coroutine execution is completed.

## Implementations of `Continuation`

Since `Continuation` is an interface, there must be implementations. Functionally, they can be roughly divided into three categories (my own summary; I haven’t studied them very deeply, so it might not be completely accurate):

1. Suspending functions and suspending lambdas are automatically compiled by the compiler into subclasses of `ContinuationImpl`. The inheritance chain looks like this: `ContinuationImpl` -> `BaseContinuationImpl` -> `Continuation`. The `resumeWith` of this kind of continuation is responsible for actually executing the code we wrote; these are the ones that do the real work (in fact, our code is placed in `invokeSuspend()`, and `BaseContinuationImpl`’s `resumeWith` calls `invokeSuspend()`, which is basically an application of the Template Method pattern).
2. Coroutine builders such as `launch` and `async`. The `Job` and `Deferred` instances they return are `StandaloneCoroutine` (or `LazyStandaloneCoroutine`) and `DeferredCoroutine` (or `LazyDeferredCoroutine`), respectively. These classes all inherit from `AbstractCoroutine`, and `AbstractCoroutine` implements `Continuation`, so these are also implementations of `Continuation`. Their main responsibility is to manage coroutine lifecycle, coroutine primitives, and parent–child relationships between coroutines. Their `resumeWith` is also for these purposes, usually used to implement state changes (for example, when the coroutine finishes execution, the state becomes `Completed`). They are basically the last link in the coroutine continuation chain.
3. Since `Continuation` is an interface with a single method, it’s very flexible: you can use delegation, decorator patterns, etc. The third kind of implementation mainly revolves around `resumeWith`, adding functionality to a `Continuation`. For example, the `SafeContinuation` mentioned above belongs to this category. Many heavyweight coroutine features are implemented this way. For instance, `DispatchedContinuation` works with dispatchers to run coroutines on different threads; its `resumeWith` implementation mainly posts the delegated continuation to another thread. Another example is `CancellableContinuation`, which checks in `resumeWith` whether the coroutine has been cancelled; this is why Kotlin coroutines can be cancelled at suspension points. In this type of implementation, the wrapped continuation is usually named `delegate`, whereas in the first two categories it is named `completion`, because the former is a proxy and the latter is a part of the coroutine continuation chain.

Let’s briefly look at the first category (because it’s the simplest). The compiler compiles suspending functions and suspending lambdas into a state machine. This state machine contains a `label` and all local variables in the function. The compiler also splits the function body by suspension points (calls to suspending functions) and numbers these segments in order. When the function reaches a suspension point, it updates the `label` in the state machine to the next number, saves all local variables, and then suspends. When resuming, `invokeSuspend()` (which is called from `resumeWith`) calls itself again, restores local variables and the `label` from the state machine, and jumps directly to the segment belonging to that `label`. For the concrete implementation of this state machine, see reference #4, which explains it very clearly.

## Interceptors and Dispatch

The coroutine context (`CoroutineContext`) is defined directly on the `Continuation` interface and belongs to the lowest-level mechanisms. You can think of it simply as a Map with key–value properties. The actual implementation uses type-safe keys and an immutable design (similar to `String` being immutable). Usually, all coroutines under the same Coroutine Scope share a context. The coroutine context itself is not special; we can create our own key–value pairs in the context and use them in business logic, similar to how `ThreadLocal` relates to `Thread`, or `Attribute` relates to a Servlet `Request`.

Of course, some keys in the context are special. Interceptors are one of the mechanisms provided directly for the first category of `Continuation`. Interceptors are implemented on top of the coroutine context, and their key is `ContinuationInterceptor`. `ContinuationImpl` only provides an `intercepted()` method, which calls the interceptor stored in the current context, giving the interceptor a chance to wrap the continuation by calling the interceptor interface `fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>` and return a proxied `Continuation` object. Almost all primitive suspending functions (as long as their names don’t contain `Unintercepted`) call `intercepted()` inside to ensure this mechanism is honored.

Coroutine dispatchers (`CoroutineDispatcher`) are implemented on top of interceptors. `CoroutineDispatcher` implements the `ContinuationInterceptor` interface, and its `interceptContinuation()` implementation returns a `DispatchedContinuation` instance (the third category we mentioned above), so the actual dispatch happens inside its `resumeWith`. The commonly used `Dispatchers.IO` and `Dispatchers.Default` are instances of subclasses of `CoroutineDispatcher`.

One question is: since a dispatcher is implemented on top of an interceptor, and an interceptor is implemented on top of the context, for the first category of `Continuation` implementations, when they are first created they have no context at all. Who assigns a context to them? The answer lies in the constructor of `ContinuationImpl`:

{{< codeblock ContinuationImpl.kt kotlin "https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/jvm/src/kotlin/coroutines/jvm/internal/ContinuationImpl.kt" >}}
internal abstract class ContinuationImpl(
    completion: Continuation<Any?>?,
    private val _context: CoroutineContext?
) : BaseContinuationImpl(completion) {
    constructor(completion: Continuation<Any?>?) : this(completion, completion?.context)
{{< /codeblock >}}

`ContinuationImpl` copies the context object from the passed-in `completion`. And since the constructor of `ContinuationImpl` is called before every suspending function and suspending lambda executes, the context is propagated along the entire coroutine continuation chain.

So where does the context of the very first `completion` come from? The answer is the second category of `Continuation` implementations mentioned above. When they construct their instances, they create a default context, merge it with the context argument passed to the coroutine builder, and then pass it as `completion` to the `ContinuationImpl` that the builder’s `block` is compiled into.

## Conclusion

The content of this article is actually not very deep. It doesn’t touch on topics such as lifecycle, parent–child relationships between coroutines, or coroutine cancellation (the second and third categories of implementations), because I haven’t studied those in depth. This is related to where my interests lie: I’m more interested in the most fundamental concepts and mechanisms, and I pay extra attention to problems that my existing knowledge can’t explain. Once you grasp the basics, the upper layers of the implementation become more flexible, and advanced topics can be explored later when real-world usage raises questions.

To be honest, I’m somewhat familiar with coroutine implementations in various languages, including C#, JavaScript, Python, and Rust, but none of them in great depth. I often get hung up on issues of execution order and call stacks. This deep dive into Kotlin coroutines helps me fill in a long-standing gap, so I can understand coroutines more comprehensively in the future.