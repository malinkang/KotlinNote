# 【译】Koltin协程：First things first

[原文](https://medium.com/androiddevelopers/coroutines-first-things-first-e6187bf3bb21)

{% hint style="success" %}
This series of blog posts goes in-depth into cancellation and exceptions in Coroutines. Cancellation is important for avoiding doing more work than needed which can waste memory and battery life; proper exception handling is key to a great user experience. As the foundation for the other 2 parts of the series ([part 2: cancellation](https://medium.com/androiddevelopers/cancellation-in-coroutines-aa6b90163629), [part 3: exceptions](https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c)), it’s important to define some core coroutine concepts such as `CoroutineScope`, `Job` and `CoroutineContext` so that we all are on the same page.
{% endhint %}

这一系列的博客文章深入讨论了`Kotlin`协程中的取消和异常。为了避免浪费内存和电池寿命，协程的取消至关重要；正确的异常处理是获得良好用户体验的关键。作为本系列其他两部分([第2部分：取消](https://juejin.cn/post/6844904121464520711)，[第3部分：异常](https://juejin.cn/post/6844904133976129543))的基础，这篇博客定义一些协程的核心概念，如协程作用域、Job和协程上下文等。

{% hint style="success" %}
If you prefer video, check out this talk from KotlinConf’19 by [Florina Muntenescu](https://medium.com/u/d5885adb1ddf?source=post\_page-----e6187bf3bb21-----------------------------------) and I:
{% endhint %}

如果你喜欢视频，可以看看KotlinConf'19的这个演讲，作者是 Florina Muntenescu 和我。

{% embed url="https://youtu.be/w0kfnydnFWI" %}

## CoroutineScope <a href="#5130" id="5130"></a>

{% hint style="success" %}
A [**`CoroutineScope`**](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/) keeps track of any coroutine you create using [`launch`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) or [`async`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) (these are extension functions on `CoroutineScope`). The ongoing work (running coroutines) can be canceled by calling `scope.cancel()` at any point in time.
{% endhint %}

_`CoroutineScope` _ 会跟踪使用_`launch`_或_`async` _ 创建的任何协程(这些是_`CoroutineScope`_的扩展函数)，并且可以在任何时间点通过调用_`scope.cancel()`_ 取消正在此上下文中进行的协程。

{% hint style="success" %}
You should create a `CoroutineScope` whenever you want to start and control the lifecycle of coroutines in a particular layer of your app. In some platforms like Android, there are KTX libraries that already provide a `CoroutineScope` in certain lifecycle classes such as [`viewModelScope`](https://developer.android.com/reference/kotlin/androidx/lifecycle/package-summary#\(androidx.lifecycle.ViewModel\).viewModelScope:kotlinx.coroutines.CoroutineScope) and [`lifecycleScope`](https://developer.android.com/reference/kotlin/androidx/lifecycle/package-summary#lifecyclescope).
{% endhint %}

无论何时你想在你的APP中启动并控制一个协程的生命周期时，都应该创建一个`CoroutineScope` __ 对象，在`Android`平台，已有`KTX`库提供了某些具有生命周期的类的 _ `CoroutineScope` _ ，如 _ `viewModelScope` _ 和 _ `lifecycleScope` _ 。\


{% hint style="success" %}
When creating a `CoroutineScope` it takes a `CoroutineContext` as a parameter to its constructor. You can create a new scope & coroutine with the following code:
{% endhint %}

&#x20;在创建 _ `CoroutineScope` _ 时，它将一个 _ `CoroutineContext` _ 对象作为构造参数，你可以使用如下代码创建新的协程作用域并使用它启动一个协程：

```kotlin
// Job and Dispatcher are combined into a CoroutineContext which
// will be discussed shortly
val scope = CoroutineScope(Job() + Dispatchers.Main)
val job = scope.launch {
    // new coroutine
}
```

## Job <a href="#7adc" id="7adc"></a>

{% hint style="success" %}
A `Job` is a handle to a coroutine. For every coroutine that you create (by `launch` or `async`), it returns a [**`Job`**](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-job/index.html) instance that uniquely identifies the coroutine and manages its lifecycle. As we saw above, you can also pass a `Job` to a `CoroutineScope` to keep a handle on its lifecycle.
{% endhint %}

Job是协程的句柄，对于您通过 _launch_ 或 _async_ 启动的每个协程，它都会返回一个Job实例，该实例唯一地标识了此协程并管理其生命周期。正如上面的代码，您还可以将Job对象传递给 _CoroutineScope_ 来保持对此协程生命周期的控制。

## CoroutineContext <a href="#c6b8" id="c6b8"></a>

{% hint style="success" %}
The [**`CoroutineContext`**](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/index.html) is a set of elements that define the behavior of a coroutine. It’s made of:

* `Job` — controls the lifecycle of the coroutine.
* [`CoroutineDispatcher`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html) — dispatches work to the appropriate thread.
* [`CoroutineName`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-name/index.html) — name of the coroutine, useful for debugging.
* [`CoroutineExceptionHandler`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-exception-handler/index.html) — handles uncaught exceptions, will be covered in Part 3 of the series.
{% endhint %}

_CoroutineContext_ 是一组定义协程行为的元素。它由：

* _Job_ — 控制协程的生命周期。
* 协程调度器（ _CoroutineDispatcher_ ）— 将协程分配到适当的线程。
* _CoroutineName_ — 协程的名称，对调试很有帮助。
* _CoroutineExceptionHandler_ — 处理未捕获的异常，将在本系列的第3部分中介绍。

{% hint style="success" %}
What’s the `CoroutineContext` of a new coroutine? We already know that a new instance of `Job` will be created, allowing us to control its lifecycle. The rest of the elements will be inherited from the `CoroutineContext` of its parent (either another coroutine or the `CoroutineScope` where it was created).
{% endhint %}

当我们创建一个新的协程时，它的协程上下文都有什么呢？我们已经知道协程会返回一个新的_Job_ 实例从而允许我们控制它的生命周期，而其余的元素将从此协程的父元素（父协程或创建它的协程作用域）继承。

{% hint style="success" %}
Since a `CoroutineScope` can create coroutines and you can create more coroutines inside a coroutine, an implicit _**task hierarchy**_ is created. In the following code snippet, apart from creating a new coroutine using the `CoroutineScope`, see how you can create more coroutines inside a coroutine:
{% endhint %}

由于协程作用域可以创建协程，并且我们可以在协程中创建更多的协程，因此会形成一个隐式的**任务层次**结构，在下面的代码中，除了使用协程作用域（ _CoroutineScope_ ）创建一个新的协程外，还可以看看如何在协程中创建更多的协程：

```kotlin
val scope = CoroutineScope(Job() + Dispatchers.Main)
val job = scope.launch {
    // New coroutine that has CoroutineScope as a parent
    val result = async {
        // New coroutine that has the coroutine started by 
        // launch as a parent
    }.await()
}
```

{% hint style="success" %}
The root of that hierarchy is usually the `CoroutineScope`. We could visualise that hierarchy as follows:
{% endhint %}

这种层次结构的根通常为最上层的协程作用域（ _CoroutineScope_ ）。我们可以把该层次结构可视化如下：

![Coroutine在一个任务层次中执行。父级可以是CoroutineScope，也可以是另一个coroutine。](<../.gitbook/assets/image (10).png>)

## Job lifecycle <a href="#5da7" id="5da7"></a>

{% hint style="success" %}
A `Job` can go through a set of states: New, Active, Completing, Completed, Cancelling and Cancelled. While we don’t have access to the states themselves, we can access properties of a Job: `isActive`, `isCancelled` and `isCompleted`.
{% endhint %}

_Job_ 经历以下一系列状态：新建、活动、正在完成、已完成、正在取消和已取消状态。虽然我们无法访问状态本身，但可以访问_Job_ 的属性: _isActive_ 、_isCancelled_ 和 _isCompleted_ 。

![Job生命周期](<../.gitbook/assets/image (12).png>)

{% hint style="success" %}
If the coroutine is in an active state, the failure of the coroutine or calling `job.cancel()` will move the job in the Cancelling state (`isActive = false`, `isCancelled = true`). Once all children have completed their work the coroutine will go in the Cancelled state and `isCompleted = true`.
{% endhint %}

如果协程处于活动状态，则协程失败或调用`job.cancel()`方法将使Job处于取消状态( `isActive = false, iscancel = true` )。一旦所有的子协程完成了他们的工作，协程将进入取消状态并且 `isCompleted = true` 。

## 关于父级CoroutineContext的解释

{% hint style="success" %}
In the task hierarchy, each coroutine has a parent that can be either a `CoroutineScope` or another coroutine. However, the resulting parent `CoroutineContext` of a coroutine can be different from the `CoroutineContext` of the parent since it’s calculated based on this formula:

**Parent context** = Defaults + inherited `CoroutineContext` + arguments
{% endhint %}

在协程的任务层次结构中，每个协程都有一个父级，可以是一个协程作用域，也可以是另一个协程。然而，协程继承的父级协程上下文可能不同于父级本身的上下文，因为协程上下文是基于这个公式计算的：

> 父级上下文 = 默认值 + 继承的上下文 + 参数

{% hint style="success" %}
Where:

* Some elements have **default** values: `Dispatchers.Default` is the default of `CoroutineDispatcher` and `“coroutine”` the default of `CoroutineName`.
* The **inherited `CoroutineContext`** is the `CoroutineContext` of the `CoroutineScope` or coroutine that created it.
* **Arguments** passed in the coroutine builder will take precedence over those elements in the inherited context.
{% endhint %}

* 某些元素具有默认值，如 `Dispatchers.Default` 是协程调度器（ _CoroutineDispatcher_ ）的默认值，“coroutine”是 _CoroutineName_ 的默认值。
* 协程会继承创建它的协程作用域或协程的 _CoroutineContext_ 。
* 在协程构建器中传递的参数会优先于继承的 _CoroutineContext_ 中的元素。

{% hint style="success" %}
_**Note**_: `CoroutineContext`s can be combined using the `+` operator. As the `CoroutineContext` is a set of elements, a new `CoroutineContext` will be created with the elements on the right side of the plus overriding those on the left. E.g. `(Dispatchers.Main, “name”) + (Dispatchers.IO) = (Dispatchers.IO, “name”)`
{% endhint %}

注意： _CoroutineContext_ 可以使用+运算符组合，由于 _CoroutineContext_ 是一组元素，因此会将加号右边的元素覆盖左侧相同的元素以创建一个新的 _CoroutineContext_ ，如 `(Dispatchers.Main, “name”) + (Dispatchers.IO) = (Dispatchers.IO, “name”)`\


![由scope创建的协程，在它的CoroutineContext中至少拥有这些元素，CoroutineName的值为灰色原因是它来自默认值](<../.gitbook/assets/image (13).png>)

{% hint style="success" %}
Now that we know what’s the parent `CoroutineContext` of a new coroutine, its actual `CoroutineContext` will be:

> **New coroutine context** = parent `CoroutineContext` + `Job()`
{% endhint %}

现在我们知道了一个新的协程的父级协程上下文是什么，那么它实际的协程上下文将是：

> 新协程的上下文 = 父级CoroutineContext + Job()

{% hint style="info" %}
If with the `CoroutineScope` shown in the image above we create a new coroutine like this:
{% endhint %}

如果使用上图的scope创建一个新协程，就像这样：

```kotlin
val job = scope.launch(Dispatchers.IO) {
    // new coroutine
}
```

{% hint style="success" %}
What’s the parent `CoroutineContext` of that coroutine and its actual `CoroutineContext`? See the solution in the image below!
{% endhint %}

那么这个新协程的协程上下文和父级协程上下文将是什么呢？请参见下图的答案！

![父级上下文中的Job永远也不会与新协程的上下文的Job是同一实例，因为新协程始终会返回一个新的Job实例](<../.gitbook/assets/image (11).png>)

{% hint style="success" %}
The resulting parent `CoroutineContext` has `Dispatchers.IO` instead of the scope’s `CoroutineDispatcher` since it was overridden by the argument of the coroutine builder. Also, check that the `Job` in the parent `CoroutineContext` is the instance of the scope’s `Job` (red color), and a new instance of `Job` (green color) has been assigned to the actual `CoroutineContext` of the new coroutine.
{% endhint %}

我们看到父级上下文中有 `Dispatchers.IO` ，而不是声明 _scope_ 时我们传入的`Dispatchers.Main` ，因为它被协程构建器 _launch_ 的参数覆盖了。另外，请注意父级上下文中的Job实例是scope对象的Job，而新协程的实际Job实例是被重新分配的(绿色)。\


{% hint style="success" %}
As we will see in Part 3 of the series, a `CoroutineScope` can have a different implementation of `Job` called `SupervisorJob` in its `CoroutineContext` that changes how the `CoroutineScope` deals with exceptions. Therefore, a new coroutine created with that scope can have `SupervisorJob` as a parent `Job`. However, when the parent of a coroutine is another coroutine, the parent `Job` will always be of type `Job`.
{% endhint %}

我们将在本系列的第三部分看到，协程上下文中有一个叫 _SupervisorJob_ 的 _Job_ 实现， _SupervisorJob_ 改变了 _CoroutineScope_ 处理异常的方式。因此，用该`scope`创建的新的 coroutine 可以将 SupervisorJob 作为父Job。然而，当一个协程的父级是协程时，父级Job将始终是Job类型的。

{% hint style="success" %}
Now that you know the basics of coroutines, start learning more about cancellation and exceptions in coroutines with Part II and Part III of this series:
{% endhint %}

现在你已经知道了协程的基础知识，可以通过本系列的第二部分和第三部分开始学习更多关于协程中的取消和异常。

{% embed url="https://medium.com/androiddevelopers/cancellation-in-coroutines-aa6b90163629" %}

{% embed url="https://medium.com/androiddevelopers/exceptions-in-coroutines-ce8da1ec060c" %}

{% embed url="https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad" %}
