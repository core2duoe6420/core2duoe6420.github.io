---
title: "Kotlin Coroutine踩坑记录"
date: 2023-10-03
slug: kotlin-coroutine-mysteries
hide_site_title: true
tags:
- kotlin
---

在项目中应用Kotlin Coroutine一年多了，虽然用到的都是一些非常基本的功能，但是也踩了不少坑，本文将这些坑记录下来做个笔记。

<!--more-->

## `runBlocking()`死锁之谜

看这样一段代码：

```kotlin
val executor = newFixedThreadPoolContext(1, "TEST")
runBlocking(executor) {
    runBlocking(executor) { }
}
```

这段代码会产生死锁。造成死锁的原因是以下两个因素叠加：

1. `runBlocking()`不是一个suspend函数，因此里层的`runBlocking`在执行时并不会让出线程，而是阻塞线程直到它创建的coroutine执行完毕
2. 调用`runBlocking()`时指定了`executor`为调度器，那么就会把里层`runBlocking`所生成的coroutine作为task放入`executor`的队列里等待执行。而`executor`的唯一线程正在执行外层`runBlocking`
生成的coroutine，由于因素1的关系，它根本没有机会去执行队列里的任务，从而造成死锁

打破以上两个因素的任意一个都可以避免死锁。我们先来看因素2。如果在调用里层`runBlocking`的时候直接写`runBlocking { }`，那么就不会死锁，
因为里层`runBlocking`会在`executor`的那个线程上创建一个`eventLoop`，然后用这个`eventLoop`去执行它生成的coroutine。

再来看因素1，如果在里层调用其它的coroutine创建函数，例如`async(executor) {}`，`launch(executor) {}`，或者是创建scope coroutine的函数，如`withContext(executor) {}`，
`coroutineScope {}`，因为这些都是非阻塞或是suspend函数，它们会让出线程，使得`executor`有机会去执行队列里的任务，因此也不会死锁。

那么到这里关于死锁的原因已经解释清楚了。读者可能会认为这个例子非常简单，最多用于教学，在实际使用中不会遇到`runBlocking()`嵌套的情况，也不会遇到调度器只有一个线程的情况。然而我们踩的坑就是在生产环境中遇到的。
当时的情况是，一旦我们尝试去获取一个列表，并且请求的数量大于64的时候，程序就会卡死，再也做不了其它任何事情。而64正好是`Dispatcher.IO`的默认线程上限。

我们知道`runBlocking()`应该只在需要桥接非suspend函数和suspend函数时才使用，那么一旦进入了suspend的领域，之后一直用suspend函数不就好了吗，就不会导致`runBlocking`嵌套了。实际情况是，
当需要用到一些针对Java开发的包，并且它提供的接口需要函数输入的时候，由于包本身就不是针对的Kotlin开发的，自然不支持suspend函数，就必须使用`runBlocking()`进行桥接。比如我们想用Guava的`Cache`接口，它提供的
`get`的接口是这样的：`V get(K key, Callable<? extends V> loader)`，这第二个参数就必须提供一个普通的函数，不能是suspend函数。遇到这种情况，最好的办法还是寻找替代品，例如我们用了`cache4k`替代了Guava的`Cache`。
如果实在找不到可以替代的包，那么应该生成一个独立的Dispatcher，专门给这里的`runBlocking()`使用，例如`Dispatchers.IO.limitedParallelism(n)`。

## `coroutineContext`作用域之谜

在任何suspend函数里我们都能直接访问一个`coroutineContext`的值，它来自于编译器提供的从`Continuation`中提取的coroutine context：

```kotlin
public interface Continuation<in T> {
    /**
     * The context of the coroutine that corresponds to this continuation.
     */
    public val context: CoroutineContext

    ...
}
```

然而在`CoroutineScope`接口里也有一个同样名为`coroutineContext`的属性：

```kotlin
public interface CoroutineScope {
    public val coroutineContext: CoroutineContext
}
```

当这两个值同时出现在一个作用域里时，`CoroutineScope`接口提供的`coroutineContext`胜出。所以以下代码会打印`a`而不是`b`

```kotlin
suspend fun runInNewContext(block: suspend () -> Unit) {
    withContext(CoroutineName("b")) {
        block()
    }
}

fun main() {
    runBlocking(CoroutineName("a")) {
        runInNewContext {
            println(coroutineContext[CoroutineName]?.name)
        }
    }
}
```

这里`coroutineContext`访问的是`runBlocking`参数中`block: suspend CoroutineScope.() -> T`，作为receiver的`CoroutineScope`提供的`coroutineContext`。

要想让上面的代码打印出`b`，将`coroutineContext`替换成`currentCoroutineContext()`即可。

这只是一个小问题，只是头一次遇到的时候搞不清楚还是挺让人懵逼的。

## `Exception`调包之谜

来看一段单元测试代码：

```kotlin
@Test
fun test() {
    val expected = Exception("test")
    val actual = assertThrows<Exception> {
        runBlocking {
            coroutineScope {
                throw expected
            }
        }
    }
    assertEquals(expected, actual)
}
```

我们使用Gradle + JUnit 5运行的单元测试，从直觉上看这段代码应该通过测试，没有问题才对。实际情况是测试失败，`expected != actual`，并且`epected`变成了`actual`的`cause`，也即`actual.cause == expected`。

这非常让人疑惑。后来通过在自定义的`Exception`构造函数里打断点，发现确实被实例化了两次，第二次实例化是因为调用了一个`recoverStackTrace()`的函数，于是顺藤摸瓜找到了[Stacktrace recovery](https://github.com/Kotlin/kotlinx.coroutines/blob/master/docs/topics/debugging.md#stacktrace-recovery)。

这应该是一个方便调试的机制，但是我并没有从例子里看出有什么大区别，先不求甚解以后再说了。

文档里也写了如何关闭这一机制，对于我们上面用Gradle写的单元测试而言，需要将所需的JVM参数写在`build.gradle.kts`里：

```kotlin
tasks.test {
    useJUnitPlatform()
    allJvmArgs = listOf(
        "-Dkotlinx.coroutines.stacktrace.recovery=false",
    )
}
```

## `ResultSet`关闭之谜

我们的项目使用Jetbrains官方为Kotlin开发的ORM框架Exposed。然而JDBC天生和coroutine不搭，因为JDBC提供的都是阻塞API。Exposed可以说是强行提供了一套函数以适配coroutine和suspend函数，
包括`newSuspendedTransaction()` `suspendedTransaction()` `suspendedTransactionAsync()`。

当我们无意中在suspended transaction里使用了`async()`并在其中进行了数据库操作之后，就有概率遇到以下问题：

![stracktrace](https://res.cloudinary.com/core2duoe6420/image/upload/v1696341587/posts/kotlin-coroutine-mysteries/stacktrace_vqg5g5.png)

目前尚不清楚深层次的原因。解决方案是使用`suspendedTransactionAsync()`代替`async()`进行数据库操作，或干脆不使用异步，按序执行数据库操作。

使用`suspendedTransactionAsync()`的问题在于无论如何它都会新建一个transaction（根据JDBC的原理，应该就是新拿了一条connection），这样会有潜在的与原有transaction数据不一致的问题，
总之就是不够完美。但是也别无他法了。