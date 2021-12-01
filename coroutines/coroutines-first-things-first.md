# 【译】Koltin协程：First things first

[原文](https://medium.com/androiddevelopers/coroutines-first-things-first-e6187bf3bb21)

{% hint style="success" %}
This series of blog posts goes in-depth into cancellation and exceptions in Coroutines. Cancellation is important for avoiding doing more work than needed which can waste memory and battery life; proper exception handling is key to a great user experience. As the foundation for the other 2 parts of the series ([part 2: cancellation](https://medium.com/androiddevelopers/cancellation-in-coroutines-aa6b90163629), [part 3: exceptions](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)), it’s important to define some core coroutine concepts such as `CoroutineScope`, `Job` and `CoroutineContext` so that we all are on the same page.
{% endhint %}

这一系列的博客文章深入讨论了`Kotlin`协程中的取消和异常。为了避免浪费内存和电池寿命，协程的取消至关重要；正确的异常处理是获得良好用户体验的关键。作为本系列其他两部分([第2部分：取消](https://juejin.cn/post/6844904121464520711)，[第3部分：异常](https://juejin.cn/post/6844904133976129543))的基础，这篇博客定义一些协程的核心概念，如协程作用域、Job和协程上下文等。

{% embed url="https://youtu.be/w0kfnydnFWI" %}

## CoroutineScope <a href="#5130" id="5130"></a>

{% hint style="success" %}
A [**`CoroutineScope`**](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/) keeps track of any coroutine you create using [`launch`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) or [`async`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) (these are extension functions on `CoroutineScope`). The ongoing work (running coroutines) can be canceled by calling `scope.cancel()` at any point in time.
{% endhint %}

_`CoroutineScope` _ 会跟踪使用_`launch`_或_`async` _ 创建的任何协程(这些是_`CoroutineScope`_的扩展函数)，并且可以在任何时间点通过调用_`scope.cancel()`_ 取消正在此上下文中进行的协程。

{% hint style="success" %}
You should create a `CoroutineScope` whenever you want to start and control the lifecycle of coroutines in a particular layer of your app. In some platforms like Android, there are KTX libraries that already provide a `CoroutineScope` in certain lifecycle classes such as [`viewModelScope`](https://developer.android.com/reference/kotlin/androidx/lifecycle/package-summary#\(androidx.lifecycle.ViewModel\).viewModelScope:kotlinx.coroutines.CoroutineScope) and [`lifecycleScope`](https://developer.android.com/reference/kotlin/androidx/lifecycle/package-summary#lifecyclescope).
{% endhint %}

每当我们想在我们的APP中启动并控制一个协程的生命周期时，都应该创建一个`CoroutineScope` __ 对象，在`Android`平台，已有`KTX`库提供了某些具有生命周期的类的 _ `CoroutineScope` _ ，如 _ `viewModelScope` _ 和 _ `lifecycleScope` _ 。\
