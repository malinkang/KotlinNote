# 在 Android 上使用协程（一）：Getting The Background

[原文](https://medium.com/androiddevelopers/coroutines-on-android-part-i-getting-the-background-3e0e54d20bb)

这是关于在 Android 中使用协程的一系列文章。本篇让我们先来看看协程是如何工作的以及它解决了什么问题。

## 协程解决了什么问题 ？

Kotlin 的 [Coroutines \(协程\)](https://kotlinlang.org/docs/reference/coroutines-overview.html) 带来了一种新的并发方式，在 Android 上，它可以用来简化异步代码。尽管 Kotlin 1.3 才带来稳定版的协程，但是自编程语言诞生以来，协程的概念就已经出现了。第一个使用协程的语言是发布于 1967 年的 [Simula](https://en.wikipedia.org/wiki/Simula) 。

在过去的几年中，协程变得越来越流行。现在许多流行的编程语言都加入了协程，例如 [Javascript](https://javascript.info/async-await) , [C\#](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/async/) , [Python](https://docs.python.org/3/library/asyncio-task.html) , [Ruby](https://ruby-doc.org/core-2.1.1/Fiber.html) , [Go](https://tour.golang.org/concurrency/1) 等等。Kotlin 协程基于以往构建大型应用中已建立的一些概念。

在安卓中，协程很好的解决了两个问题：

1. **耗时任务**，运行时间过长阻塞主线程
2. **主线程安全**，允许你在主线程中调用任意 suspend\(挂起\) 函数

下面让我们深入了解协程如何帮助我们构建更干净的代码！

## 耗时任务

获取网页，和 API 进行交互，都涉及到了网络请求。同样的，从数据库读取数据，从硬盘中加载图片，都涉及到了文件读取。这些就是我们所说的耗时任务，App 不可能特地暂停下来等待它们执行完成。

和网络请求相比，很难具体的想象现代智能手机执行代码的速度有多快。`Pixel 2` 的一个 CPU 时钟周期不超过 `0.0000000004` 秒，这是一个对人类来说很难理解的一个数字。但是如果你把一次网络请求的耗时想象成一次眨眼，大概 0.4 s，这就很好理解 CPU 执行的到底有多快了。在一次眨眼的时间内，或者一次较慢的网络请求，CPU 可以执行超过一百万次时钟周期。

在 Android 中，每个 app 都有一个主线程，负责处理 UI\(例如 View 的绘制\)和用户交互。如果在主线程中处理过多任务，应用将会变得卡顿，随之带来了不好的用户体验。任何耗时任务都不应该阻塞主线程，为了避免在主线程中进行网络请求，一种通用的模式是使用 `CallBack`\(回调\)，它可以在将来的某一时间段回调进入你的代码。使用回调访问 `developer.android.com` 如下所示：

```kotlin
class ViewModel: ViewModel() {
   fun fetchDocs() {
       get("developer.android.com") { result ->
           show(result)
       }
    }
}
```

尽管 `get()` 方法是在主线程调用的，但它会在另一个线程中进行网络请求。一旦网络请求的结果可用了，回调就会在主线程中被调用。这是处理耗时任务的一种好方式，像 [Retrofit](https://square.github.io/retrofit/) 就可以帮助你进行网络请求并且不阻塞主线程。

## 使用协程处理耗时任务

用协程来处理耗时任务可以简化代码。以上面的 `fetchDocs()` 方法为例，我们使用协程来重写之前的回调逻辑。

```kotlin
// Dispatchers.Main
suspend fun fetchDocs() {
    // Dispatchers.IO
    val result = get("developer.android.com")
    // Dispatchers.Main
    show(result)
}
// look at this in the next section
suspend fun get(url: String) = withContext(Dispatchers.IO){/*...*/}
```

上面的代码会阻塞主线程吗？它是如何在不暂停等待网络请求或者阻塞主线程的情况下得到 `get()` 的返回值的？事实证明，Kotlin 协程提供了一种永远不会阻塞主线程的代码执行方式。

协程添加了两个操作来构建一些常规功能。除了 `invoke(or call)` 和 `return` ，它额外添加了 `suspend(挂起)` 和 `resume(恢复)`。

* **suspend** —— 挂起当前协程的执行，保存所有局部变量
* **resume** —— 从被挂起协程挂起的地方继续执行

在 Kotlin 中，通过给函数添加 `suspend` 关键字来实现此功能。你只能在挂起函数中调用挂起函数，或者通过协程构造器，例如 `launch` ，来开启一个新的协程。

> 挂起和恢复共同工作来替代回调。

在上面的例子中，`get()` 方法在进行网络请求之前会挂起协程，它也负责进行网络请求。然后，当网络请求结束时，它仅仅只需要恢复之前挂起的协程，而不是调用回调函数来通知主线程。

![](../.gitbook/assets/image%20%287%29.png)

> 当主线程上的所有协程都被挂起，它就有时间做其他事情了。

即使我们直接顺序书写代码，看起来就像是会导致阻塞的网络请求一样，但是协程会按我们所希望的那样执行，不会阻塞主线程。

下面，让我们看看协程是如何做到主线程安全的，并且探索一下 `disaptchers(调度器)`。  


## 协程的主线程安全

在 Kotlin 协程中，编写良好的挂起函数在主线程中调用总是安全的。无论挂起函数做了什么，总是应该允许任何线程调用它们。

但是，在 Android 应用中，我们如果把很多工作都放在主线程做会导致 APP 运行缓慢，例如网络请求，JSON 解析，读写数据库，甚至是大集合的遍历。它们中任何一个都会导致应用卡顿，降低用户体验。所以它们不应该运行在主线程。

使用 suspend 并不意味着告诉 Kotlin 一定要在后台线程运行函数。值得一提的是，协程经常运行在主线程。事实上，当启动一个用于响应用户事件的协程时，使用 [Dispatchers.Main.immediate](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-main-coroutine-dispatcher/immediate.html) 是一个好主意。

> 协程也会运行在主线程，suspend 并不一定意味着后台运行。

为了让一个函数不会使主线程变慢，我们可以告诉 Kotlin 协程使用 `Default` 或者 `IO` 调度器。在 Kotlin 中，所有的协程都需要使用调度器，即使它们运行在主线程。协程可以挂起自己，而调度器就是用来告诉它们如何恢复运行的。

为了指定协程在哪里运行，Kotlin 提供了 [Dispatchers](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/) 来处理线程调度。

```text
+-----------------------------------+
|         Dispatchers.Main          |
+-----------------------------------+
| Main thread on Android, interact  |
| with the UI and perform light     |
| work                              |
+-----------------------------------+
| - Calling suspend functions       |
| - Call UI functions               |
| - Updating LiveData               |
+-----------------------------------+

+-----------------------------------+
|          Dispatchers.IO           |
+-----------------------------------+
| Optimized for disk and network IO |
| off the main thread               |
+-----------------------------------+
| - Database*                       |
| - Reading/writing files           |
| - Networking**                    |
+-----------------------------------+

+-----------------------------------+
|        Dispatchers.Default        |
+-----------------------------------+
| Optimized for CPU intensive work  |
| off the main thread               |
+-----------------------------------+
| - Sorting a list                  |
| - Parsing JSON                    |
| - DiffUtils                       |
+-----------------------------------+
```

* [Room](https://developer.android.com/topic/libraries/architecture/room) 在你使用 [挂起函数](https://medium.com/androiddevelopers/room-coroutines-422b786dc4c5) 、[RxJava](https://medium.com/androiddevelopers/room-rxjava-acb0cd4f3757) 、[LiveData](https://developer.android.com/topic/libraries/architecture/livedata#use_livedata_with_room) 时自动提供主线程安全。
* [Retrofit](https://square.github.io/retrofit/) 和 [Volley](https://developer.android.com/training/volley) 等网络框架一般自己管理线程调度，当你使用 Kotlin 协程的时候不需要再显式保证主线程安全。

继续上面的例子，让我们使用调度器来定义 get 函数。在 get 函数的方法体内使用 `withContext(Dispatchers.IO)` 定义一段代码块，这个代码块将在调度器 `Dispatchers.IO` 中运行。方法块中的任何代码总是会运行在 IO 调度器中。由于 `withContext` 本身就是一个挂起函数，所以它通过协程提供了主线程安全。

```kotlin
// Dispatchers.Main
suspend fun fetchDocs() {
    // Dispatchers.Main
    val result = get("developer.android.com")
    // Dispatchers.Main
    show(result)
}
// Dispatchers.Main
suspend fun get(url: String) =
    // Dispatchers.IO
    withContext(Dispatchers.IO) {
        // Dispatchers.IO
        /* perform blocking network IO here */
    }
    // Dispatchers.Main
}
```

通过协程，你可以细粒度的控制线程调度，因为 `withContext` 让你可以控制任意一行代码运行在什么线程上，而不用引入回调来获取结果。你可将其应用在很小的函数中，例如数据库操作和网络请求。所以，比较好的做法是，使用 `withContext` 确保每个函数在任意调度器上执行都是安全的，包括 `Main`，这样调用者在调用函数时就不需要考虑应该运行在什么线程上。

> 编写良好的挂起函数被任意线程调用都应该是安全的。

保证每个挂起函数主线程安全无疑是个好主意，如果它涉及到任何磁盘，网络，或者 CPU 密集型的任务，请使用 withContext 来确保主线程调用是安全的。这也是基于协程的库所遵循的设计模式。如果你的整个代码库都遵循这一原则，你的代码将会变得更加简单，线程问题和程序逻辑也不会再混在一起。协程可以自由的从主线程启动，数据库和网络请求的代码会更简单，且能保证用户体验。

## withContext 的性能

对于提供主线程安全性，withContext 与回调或 RxJava 一样快。在某些情况下，甚至可以使用协程上下文 withContext 来优化回调。如果一个函数将对数据库进行10次调用，那么您可以告诉 Kotlin 在外部的 withContext 中调用一次切换。尽管数据库会重复调用 withContext ，但是他它将在同一个调度器下，寻找最快路径。此外，`Dispatchers.Default` 和 `Dispatchers.IO` 之间的协程切换已经过优化，以尽可能避免线程切换。  


## What’s next

在这篇文章中我们探索了协程解决了什么问题。协程是编程语言中一个非常古老的概念，由于它们能够使与网络交互的代码更简单，因此最近变得更加流行。

在安卓上，你可以使用协程解决两个常见问题：

* 简化耗时任务的代码，例如网络请求，磁盘读写，甚至大量 JSON 的解析
* 提供准确的主线程安全，在不会让代码更加臃肿的情况下保证不阻塞主线程

在下一篇文章中，我们将探索它们是如何适应 Android 的，以便跟踪您从屏幕开始的所有工作!（In the next post we’ll explore how they fit in on Android to keep track of all the work you started from a screen! ）  


  


