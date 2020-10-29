# 在 Android 上使用协程（二）：Getting started

[原文](https://medium.com/androiddevelopers/coroutines-on-android-part-ii-getting-started-3bff117176dd)

这是关于在 Android 中使用协程的一系列文章。本篇的重点是开始任务以及追踪已经开始的任务。

## 追踪协程

在上篇文章中，我们探索了协程擅长解决的问题。通常，协程对于下面两个常见的编程问题来说都是不错的解决方案：

1. **耗时任务**，运行时间过长阻塞主线程
2. **主线程安全**，允许你在主线程中调用任意 suspend\(挂起\) 函数

为了解决这些问题，协程基于基础函数添加了 `suspend` 和 `resume`。当特定线程上的所有协程都被挂起，该线程就可以做其他工作了。

但是，协程本身并不能帮助你追踪正在进行的任务。同时拥有并挂起数百甚至上千的协程是不可能的。尽管协程是轻量的，但它们执行的任务并不是，例如文件读写，网络请求等。

使用代码手动追踪一千个协程的确是很困难的。你可以尝试去追踪它们，并且手动保证它们最后会完成或者取消，但是这样的代码冗余，而且容易出错。如果你的代码不够完美，你将失去对一个协程的追踪，我把它称之为任务泄露。

任务泄露就像内存泄露一样，而且更加糟糕。对于已经丢失泄露的协程，除了内存消耗之外，它还会恢复自己来消耗 CPU，磁盘，甚至启动一个网络请求。

> 泄露的协程会浪费内存，CPU，磁盘，甚至发送一个不需要的网络请求。

为了避免泄露协程，Kotlin 引入了 [structured concurrency\(结构化并发\)](https://kotlinlang.org/docs/reference/coroutines/basics.html#structured-concurrency)。结构化并发集合了语言特性和最佳实践，遵循这个原则将帮助你追踪协程中的所有任务。

在 Android 中，我们使用结构化并发可以做三件事：

1. 取消不再需要的任务
2. 追踪所有正在进行的任务
3. 协程失败时的错误信号

让我们深入探讨这几点，来看看结构化并发是如何帮助我们避免丢失对协程的追踪以及任务泄露。

## 通过作用域取消任务

在 Kotlin 中，协程必须运行在 `CoroutineScope` 中。`CoroutineScope` 会追踪你的协程，即使协程已经被挂起。不同于上一篇文章中说过的 `Dispatchers`，它实际上并不执行协程，它仅仅只是保证你不会丢失对协程的追踪。

为了保证所有的协程都被追踪到，Kotlin 不允许你在没有 `CoroutineScope` 的情况下开启新的协程。你可以把 `CoroutineScope` 想象成具有特殊能力的轻量级的 `ExecutorServicce`。它赋予你创建新协程的能力，这些协程都具备我们在上篇文章中讨论过的挂起和恢复的能力。

`CoroutineScope` 会追踪所有的协程，并且它也可以取消所有由他开启的协程。这很适合 Android 开发者，当用户离开当前页面后，可以保证清理掉所有已经开启的东西。

> CoroutineScope 会追踪所有的协程，并且它也可以取消所有由他开启的协程。

### 启动新的协程

有一点需要注意的是，你不是在任何地方都可以调用挂起函数。挂起和恢复机制要求你从普通函数切换到协程。

启动协程有两种方法，且有不同的用法：

1. 使用 [launch](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/launch.html) 协程构建器启动一个新的协程，这个协程是没返回值的
2. 使用 [async](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/async.html) 协程构建器启动一个新的协程，它允许你返回一个结果，通过挂起函数 `await` 来获取。

在大多数情况下，如何从一个普通函数启动协程的答案都是使用 `launch`。因为普通函数是不能调用 `await` 的（记住，普通函数不能直接调用挂起函数）。稍后我们会讨论什么时候应该使用 `async`。

你应该调用 `launch` 来使用协程作用域启动一个新的协程。

```kotlin
scope.launch {
    // This block starts a new coroutine
    // "in" the scope.
    //
    // It can call suspend functions
    fetchDocs()
}
```

你可以把 `launch` 想象成一座桥梁，连接了普通函数中的代码和协程的世界。在 `launch` 内部，你可以调用挂起函数，并且创建主线程安全性，就像上篇文章中提到的那样。

> Launch 是把普通函数带进协程世界的桥梁。

{% hint style="info" %}
提示：`launch` 和 `async` 很大的一个区别是异常处理。`async` 期望你通过调用 `await` 来获取结果（或异常），所以它默认不会抛出异常。这就意味着使用 `async` 启动新的协程，它会悄悄的把异常丢弃。
{% endhint %}

由于 `launch` 和 `async` 只能在 `CoroutineScope` 中使用，所以你创建的每一个协程都会被协程作用域追踪。Kotlin 不允许你创建未被追踪的协程，这样可以有效避免任务泄露。

### 在 ViewModel 中启动

如果一个 `CoroutineScope` 追踪在其中启动的所有协程，`launch` 会新建一个协程，那么你应该在何处调用 `launch` 并将其置于协程作用域中呢？还有，你应该在什么时候取消在作用域中启动的所有协程呢？  


在 Android 中，通常将 `CoroutineScope` 和用户界面相关联起来。这将帮助你避免协程泄露，并且使得用户不再需要的 `Activity` 或者 `Fragment` 不再做额外的工作。当用户离开当前页面，与页面相关联的 `CoroutineScope` 将取消所有工作。  


> 结构化并发保证当协程作用域取消，其中的所有协程都会取消。

当通过 Android Architecture Components 集成协程时，一般都是在 `ViewModel` 中启动协程。这里是许多重要任务开始工作的地方，并且你不必担心旋转屏幕会杀死协程。

为了在 `ViewModel` 中使用协程，你可以来自 `lifecycle-viewmodel-ktx:2.1.0- alpha04` 这个库的 `viewModelScope`。`viewModelScope` 即将在 [Android Lifecycle v2.1.0](https://developer.android.com/jetpack/androidx/releases/lifecycle) 发布，现在仍然是 alpha 版本。关于 `viewModelScope` 的原理可以阅读 [这篇博客](https://medium.com/androiddevelopers/easy-coroutines-in-android-viewmodelscope-25bffb605471)。既然这个库目前还是 alpha 版本，就可能会有 bug，API 也可能发生变动。如果你找到了 bug，可以在 [这里](https://issuetracker.google.com/issues?q=componentid:413132) 提交。

看一下使用的例子：

```kotlin
class MyViewModel(): ViewModel() {
    fun userNeedsDocs() {
        // Start a new coroutine in a ViewModel
        viewModelScope.launch {
            fetchDocs()
        }
    }
}
```

当 `viewModelScope` 被清除（即 `onCleared()` 被调用）时，它会自动取消由它启动的所有协程。这肯定是正确的行为，当我们还没有读取到文档，用户已经关闭了 app，我们还继续请求的话只是在浪费电量。

为了更高的安全性，协程作用域会自动传播。如果你启动的协程中又启动了另一个协程，它们最终会在同一个作用域中结束。这就意味着你依赖的库通过你的 `viewModelScope` 启动了新的协程，你就有办法取消它们了！

所以，当你需要协程和 `ViewModel` 的生命周期保持一致时，使用 `viewModelScope` 来从普通函数切换到协程。那么，由于 `viewModelScope` 会自动取消协程，编写下面这样的无限循环是没有问题的，不会造成泄露。

```kotlin
fun runForever() {
    // start a new coroutine in the ViewModel
    viewModelScope.launch {
        // cancelled when the ViewModel is cleared
        while(true) {
        delay(1_000)
        // do something every second
        }
    }
}
```

使用 `viewModelScope`，你可以确保任何工作，即使是死循环，都能在不再需要执行的时候将其取消。  


## 追踪任务

启动一个协程是没问题的，很多时候也正是这样做的。通过一个协程，进行网络请求，保存数据到数据库。

有时候，情况会稍微有点复杂。如果你想在一个协程中同时进行两个网络请求，你就需要启动更多的协程。

为了启动更多的协程，任何挂起函数都可以使用 [coroutineScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/coroutine-scope.html) 或者 [supervisorScope](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/supervisor-scope.html) 构建器来新建协程。这个 API，说实话有点让人困惑。`coroutineScope` 构建器和 `CoroutineScope` 是两个不同的东西，却只有一个字母不一样。

在任何地方启动新协程，这可能会导致潜在的任务泄露。调用者可能都不知道新协程的启动，它又如何其跟踪呢？

结构化并发帮助我们解决了这个问题。它给我们提供了一个保障，保证当挂起函数返回时，它的所有工作都已经完成。

> 结构化并发保证当挂起函数返回时，它的所有任务都已经完成。

下面是使用 `coroutineScope` 来查询文档的例子：

```text
suspend fun fetchTwoDocs() {
    coroutineScope {
        launch { fetchDoc(1) }
        async { fetchDoc(2) }
    }
}
```

在这个例子中，同时从网络读取两个文档。第一个是在由 `launch` 启动的协程中执行，它不会给调用者返回任何结果。

第二个使用的是 `async`，所以文档可以返回给调用者。这里例子有点奇怪，通常两个文档都会使用 `async`。但是我只是想向你展示你可以根据你的需求混合使用 `launch` 和 `async`。  


> coroutineScope 和 supervisorScope 让你可以安全的在挂起函数中启动协程。

尽管上面的代码没有在任何地方显示的声明要等待协程的执行完成，看起来当协程还在运行的时候，`fetDocs` 方法就会返回。

为了结构化并发和避免任务泄露，我们希望确保当挂起函数（例如 `fetchTwoDocs`）返回时，它的所有任务都已经完成。这就意味着，由 `fetchTwoDocs` 启动的所有协程都会先于它返回之前执行结束。

`Kotlin` 通过 `coroutineScope` 构建器确保 `fetchTwoDocs` 中的任务不会泄露。`coroutineScope` 构建器直到在其中启动的所有协程都执行结束时才会挂起自己。正因如此，在 `coroutineScope` 中的所有协程尚未结束之前就从 `fetchTwoDocs` 中返回是不可能的。  


## 许多许多任务

现在我们已经探索了如何追踪一个和两个协程，现在是时候来尝试追踪一千个协程了！

看一下下面的动画：

![](../.gitbook/assets/image%20%288%29.png)

这个例子展示了同时进行一千次网络请求。这在真实的代码中是不建议的，会浪费大量资源。

上面的代码中，我们在 `coroutineScope` 中通过 `launch` 启动了一千个协程。你可以看到它们是如何连接起来的。由于我们是在挂起函数中，所以某个地方的代码一定是使用了 `CoroutineScope` 来启动协程。对于这个 `CoroutineScope`，我们一无所知，它可能是 `viewModelScope` 或者定义在其他地方的 `CoroutineScope`。无论它是什么作用域，`coroutineScope` 构建器都会把它当做新建作用域的父亲。

在 `coroutineScope` 代码块中，`launch` 将在新的作用域中启动协程。当协程完成启动，这个新的作用域将追踪它。最后，一旦在 `coroutineScope` 中启动的所有协程都完成了，`loadLots` 就可以返回了。  
这里有很多事情在进行，其中最重要的就是使用 `coroutineScope` 或者 `supervisorScope`，你可以在任意挂起函数中安全的启动协程。尽管这将启动一个新协程，你也不会意外的泄露任务，因为只有所有新协程都完成了你才可以挂起调用者。

很酷的是 `coroutineScope` 可以创建子作用域。如果父作用域被取消，它会将取消动作传递给所有的新协程。如果调用者是 `viewModelScope`，当用户离开页面是，所有的一千个协程都会自动取消。多么的整洁！

在我们移步谈论异常处理之前，有必要来讨论一下 `coroutineScope` 和 `supervisorScope`。它们之间最大的不同就是，当其中任意一个子协程失败时，`coroutineScope` 会取消。所以，如果一个网络请求失败了，其他的所有请求都会立刻被取消。如果你想继续执行其他请求的话，你可以使用 `supervisorScope`，当一个子协程失败时，它不会取消其他的子协程。

## 协程失败的异常处理

在协程中，错误也是用过抛出异常来发出信号，和普通函数一样。挂起函数的异常将在 resume 的时候重新抛出给调用者。和普通函数一样，你不会被限制使用 try/catch 来处理错误，你也可以按你喜欢的方式来处理异常。

但是，有一些情况下，协程中的异常会丢失。  


```java
val unrelatedScope = MainScope()
    // example of a lost error
    suspend fun lostError() {
        // async without structured concurrency
        unrelatedScope.async {throw InAsyncNoOneCanHearYou("except")
    }
}
```

注意，上面的代码中声明了一个未经关联的协程作用域，并且未通过结构化并发启动新协程。记住我开始说过的，结构化并发集合了语言特性和最佳实践，在挂起函数中引入未经关联的协程作用并不是结构化并发的最佳实践。

上面代码中的错误会丢失，因为 `async` 认为你会调用 `await`，这时候会重新抛出异常。但是如果你没有调用 `await`，这个错误将永远被保存，静静的等待被发现。

> 结构化并发保证当一个协程发生错误，它的调用者或者作用域可以发现。

如果我们使用结构化并发写上面的代码，异常将会正确的抛给调用者。

```text
suspend fun foundError() {
    coroutineScope {
        async {
            throw StructuredConcurrencyWill("throw")
        }
    }
}
```

由于 `coroutineScope` 会等待所有子协程执行完成，所以当子协程失败时它也会知道。当 `coroutineScope` 启动的协程抛出了异常，`coroutineScope` 会将异常扔给调用者。如果使用 `coroutineScope` 代替 `supervisorScope`，当异常抛出时，会立刻停止所有的子协程。  


## 使用结构化并发

## 在这篇文章中，我介绍了结构化并发，以及在代码中配合 `ViewModel` 使用来避免任务泄露。我还谈论了它是如何让挂起函数更加简单。两者都确保在返回之前完成任务，也可以确保正确的异常处理。

我们使用非结构化并发，很容易造成意外的任务泄露，这对调用者来说是未知的。任务将变得不可取消，也不能保证异常被正确的抛出。这会导致我们的代码产生一些模糊的错误。

使用未关联的 `CoroutineScope`（注意是大写字母 C），或者使用全局作用域 `GlobalScope` ，会导致非结构化并发。只有在少数情况下，你需要协程的生命周期长于调用者的作用域时，才考虑使用非结构化并发。通常情况下，你都应该使用结构化并发来追踪协程，处理异常，拥有良好的取消机制。

如果你有非结构化并发的经验，那么结构化并发的确需要一些时间来适应。这种保障使得和挂起函数交互更加安全和简单。我们应该尽可能的使用结构化并发，因为它使得代码更加简单和易读。

在文章的开头，我列举了结构化并发帮助我们解决的三个问题：

1. 取消不再需要的任务
2. 追踪所有正在进行的任务
3. 协程失败时的错误信号

结构化并发给予我们如下保证：

1. 当作用域取消，其中的协程也会取消
2. 当挂起函数返回，其中的所有任务都已完成
3. 当协程发生错误，其调用者会得到通知

这些加在一起，使得我们的代码更加安全，简洁，并且帮助我们避免任务泄露。

## What's Next?

这篇文章中，我们探索了如何在 Android 的 ViewModel 中启动协程，以及如何使用结构化并发来优化代码。

下一篇中，我们将更多的讨论在特定情况下使用协程。  






  


