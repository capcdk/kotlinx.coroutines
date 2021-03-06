<!--- INCLUDE .*/example-([a-z]+)-([0-9a-z]+)\.kt 
/*
 * Copyright 2016-2018 JetBrains s.r.o. Use of this source code is governed by the Apache 2.0 license.
 */

// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.$$1$$2
-->
<!--- KNIT     ../core/kotlinx-coroutines-core/test/guide/.*\.kt -->
<!--- TEST_OUT ../core/kotlinx-coroutines-core/test/guide/test/CancellationTimeOutsGuideTest.kt
// This file was automatically generated from coroutines-guide.md by Knit tool. Do not edit.
package kotlinx.coroutines.guide.test

import org.junit.Test

class CancellationTimeOutsGuideTest {
--> 
## 目录

<!--- TOC -->

* [取消与超时](#cancellation-and-timeouts)
  * [取消协程的执行](#cancelling-coroutine-execution)
  * [取消协作](#cancellation-is-cooperative)
  * [使执行计算的代码可被取消](#making-computation-code-cancellable)
  * [在 finally 中释放资源](#closing-resources-with-finally)
  * [运行不能被取消的代码块](#run-non-cancellable-block)
  * [超时](#timeout)

<!--- END_TOC -->

## 取消与超时

这一部分包含了协程的取消与超时。

### 取消协程的执行

在一个长时间运行的应用程序中，你也许需要对你的后台协程进行细粒度的控制。
比如说，一个用户也许关闭了一个启动了协程的界面，那么现在协程的执行结果<!--
-->已经不再被需要了，这时，它应该是可以被取消的。
该 [launch] 函数返回了一个可以被用来取消运行中的协程的 [Job]：
 
<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">
 
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
    delay(1300L) // 延迟一段时间
    println("main: I'm tired of waiting!")
    job.cancel() // 取消该任务
    job.join() // 等待任务执行结束
    println("main: Now I can quit.")
//sampleEnd
}
``` 

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-cancel-01.kt)获得完整代码

程序执行后的输出如下：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

<!--- TEST -->

一旦 main 函数调用了 `job.cancel`，我们在其它的协程中就看不到任何输出，因为它被取消了。
这里也有一个可以使 [Job] 挂起的函数 [cancelAndJoin]<!--
-->它合并了对 [cancel][Job.cancel] 以及 [join][Job.join] 的调用。

### 取消协作

协程的取消是 _协作_ 的。一段协程代码必须协作才能被取消。
所有 `kotlinx.coroutines` 中的挂起函数都是 _可被取消的_ 。它们检查协程的取消，
并在取消时抛出 [CancellationException]。 然而，如果协程正在执行<!--
-->计算任务，并且没有检查取消的话，那么它是不能被取消的，就如如下示例<!--
-->代码所示：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (i < 5) { // 一个执行计算的循环，只是为了占用CPU
            // 每秒打印消息两次
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // 等待一段时间
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // 取消一个任务并且等待它结束
    println("main: Now I can quit.")
//sampleEnd 
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-cancel-02.kt)获得完整代码

运行示例代码，并且我们可以看到它连续打印出了 “I'm sleeping” ，甚至在调用取消后，
任务仍然执行了五次循环迭代并运行到了它结束为止。

<!--- TEST 
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm sleeping 3 ...
I'm sleeping 4 ...
main: Now I can quit.
-->

### 使执行计算的代码可被取消

我们有两种方法来使执行计算的代码可以被取消。第一种方法是定期<!--
-->调用挂起函数来检查取消。 对于这种目的 [yield] 是一个好的选择。
另一种方法是明确的检查取消状态。让我们试试第二种方法。

将前一个示例中的 `while (i < 5)` 替换为 `while (isActive)` 并重新运行它。 

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (isActive) { // 可以被取消的计算循环
            // 每秒打印消息两次
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // 等待一段时间
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // 取消该任务并等待它结束
    println("main: Now I can quit.")
//sampleEnd  
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-cancel-03.kt)获得完整代码

你可以看到，现在循环被取消了。[isActive] 是一个可以被使用在<!--
-->[CoroutineScope] 中的扩展属性。

<!--- TEST
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
-->

### 在finally中释放资源

我们通常使用如下的方法处理在被取消时抛出 [CancellationException] 的可被取消<!--
-->的挂起函数。比如说，`try {...} finally {...}` 表达式以及 Kotlin 的 `use` 函数一般在协程被取消的时候<!--
-->执行它们的终结动作：
 
 
<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">
 
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            println("I'm running finally")
        }
    }
    delay(1300L) // 延迟一段时间
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // 取消该任务并且等待它结束
    println("main: Now I can quit.")
//sampleEnd 
}
``` 

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-cancel-04.kt)获得完整代码

[join][Job.join] 和 [cancelAndJoin] 等待了所有的终结动作执行完毕， 
所以运行示例得到了下面的输出：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm running finally
main: Now I can quit.
```

<!--- TEST -->

### 运行不能被取消的代码块

在前一个例子中任何尝试在 `finally` 块中调用挂起函数的行为都会抛出
[CancellationException]，因为这里持续运行的代码是可以被取消的。通常，这并不是一个<!--
-->问题，所有良好的关闭操作（关闭一个文件、取消一个任务，或是关闭任何一种<!--
-->通信通道）通常都是非阻塞的，并且不会调用任何挂起函数。然而，在<!--
-->真实的案例中，当你需要挂起一个被取消的协程，你可以将相应的代码包装在
`withContext(NonCancellable) {...}` 中，并使用 [withContext] 函数以及 [NonCancellable] 上下文，见如下示例所示：
 
<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">
 
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val job = launch {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable) {
                println("I'm running finally")
                delay(1000L)
                println("And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
    delay(1300L) // 延迟一段时间
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // 取消该任务并等待它结束
    println("main: Now I can quit.")
//sampleEnd
}
``` 

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-cancel-05.kt)获得完整代码

<!--- TEST
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main: I'm tired of waiting!
I'm running finally
And I've just delayed for 1 sec because I'm non-cancellable
main: Now I can quit.
-->

### 超时

在实践中绝大多数取消一个协程的理由是<!--
-->它有可能超时。
当你手动追踪一个相关 [Job] 的引用并启动了一个单独的协程在<!--
-->延迟后取消追踪，这里已经准备好使用 [withTimeout] 函数来做这件事。
来看看示例代码：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    withTimeout(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }
//sampleEnd
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-cancel-06.kt)获得完整代码

运行后得到如下输出：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Exception in thread "main" kotlinx.coroutines.TimeoutCancellationException: Timed out waiting for 1300 ms
```

<!--- TEST STARTS_WITH -->

[withTimeout] 抛出了 `TimeoutCancellationException`，它是 [CancellationException] 的子类。
我们之前没有在控制台上看到堆栈跟踪信息的打印。这是因为<!--
-->在被取消的协程中 `CancellationException` 被认为是协程执行结束的正常原因。 
然而，在这个示例中我们在 `main` 函数中正确地使用了 `withTimeout`。

由于取消只是一个例外，所有的资源都使用常用的方法来关闭。
如果你需要做一些各类使用超时的特别的额外操作，可以使用类似 [withTimeout]
的 [withTimeoutOrNull] 函数，并把这些会超时的代码包装在 `try {...} catch (e: TimeoutCancellationException) {...}`
代码块中，而 [withTimeoutOrNull] 通过返回 `null` 来进行超时操作，从而替代抛出一个异常：

<div class="sample" markdown="1" theme="idea" data-min-compiler-version="1.3">

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
//sampleStart
    val result = withTimeoutOrNull(1300L) {
        repeat(1000) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
        "Done" // 在它运行得到结果之前取消它
    }
    println("Result is $result")
//sampleEnd
}
```

</div>

> 你可以点击[这里](../core/kotlinx-coroutines-core/test/guide/example-cancel-07.kt)获得完整代码

运行这段代码时不再抛出异常：

```text
I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
Result is null
```

<!--- TEST -->

<!--- MODULE kotlinx-coroutines-core -->
<!--- INDEX kotlinx.coroutines -->
[launch]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html
[Job]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html
[cancelAndJoin]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/cancel-and-join.html
[Job.cancel]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/cancel.html
[Job.join]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/join.html
[CancellationException]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-cancellation-exception/index.html
[yield]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/yield.html
[isActive]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html
[CoroutineScope]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/index.html
[withContext]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-context.html
[NonCancellable]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-non-cancellable.html
[withTimeout]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout.html
[withTimeoutOrNull]: https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/with-timeout-or-null.html
<!--- END -->
