# Kotlin协程的取消

[原文](https://medium.com/androiddevelopers/cancellation-in-coroutines-aa6b90163629)

{% hint style="success" %}
In development, as in life, we know it’s important to avoid doing more work than needed as it can waste memory and energy. This principle applies to coroutines as well. You need to make sure that you control the life of the coroutine and cancel it when it’s no longer needed — this is what structured concurrency represents. Read on to find out the ins and outs of coroutine cancellation.
{% endhint %}



作为开发者，我们通常会花费大量的时间来完善我们的应用程序。然而，当事情不尽如人意时，提供合适的用户体验也同样重要。一方面，看到应用程序崩溃对用户来说是一种很糟糕的体验；另一方面，当一个操作失败时，向用户显示正确的提示信息也是必不可少的。 正确的处理异常对用户如何看待我们的App尤为重要。在本文中，我们将解释异常时如何在协程中传播的，以及如何始终进行控制，包括处理异常的不同方法。

{% hint style="success" %}
If you prefer to see a video on this check out the talk&#x20;

[Manuel Vivo](https://medium.com/u/3b5622dd813c?source=post\_page-----aa6b90163629-----------------------------------) and I gave at KotlinConf’19 on coroutines cancellation and exceptions:
{% endhint %}


{% hint style="success" %}
> ⚠️ In order to follow the rest of the article without any problems, reading and understanding Part I of the series is required.
{% endhint %}

> ⚠️ 为了无障碍的阅读本文的其余部分，请阅读并理解本系列的第一章

## 取消正在进行的协程

{% hint style="success" %}
When launching multiple coroutines, it can be a pain to keep track of them or cancel each individually. Rather, we can rely on cancelling the entire scope coroutines are launched into as this will cancel all of the child coroutines created:
{% endhint %}

当启动多个协程时，逐个跟踪或取消它们可能会很麻烦，但是我们可以依靠取消父协程或协程作用域，因为这将取消它创建的所有协程。


```kotlin
// assume we have a scope defined for this layer of the app
val job1 = scope.launch { … }
val job2 = scope.launch { … }
scope.cancel()
```

{% hint style="success" %}
Cancelling the scope cancels its children
{% endhint %}
取消一个协程作用域将同时取消此作用域下的所有子协程。

{% hint style="success" %}
Sometimes you might need to cancel only one coroutine, maybe as a reaction to a user input. Calling job1.cancel ensures that only that specific coroutine gets cancelled and all the other siblings are not affected:
{% endhint %}

有时您可能只需要取消一个协程，调用job1.cancel()可确保仅取消特定协程，所有它的同级协程都不会受到影响。

```kotlin
// assume we have a scope defined for this layer of the app
val job1 = scope.launch { … }
val job2 = scope.launch { … }
// First coroutine will be cancelled and the other one won’t be affected
job1.cancel()
```

{% hint style="success" %}
A cancelled child doesn’t affect other siblings
{% endhint %}
被取消的子协程不会影响到其他同级的协程。

{% hint style="success" %}
Coroutines handle cancellation by throwing a special exception: CancellationException. If you want to provide more details on the cancellation reason you can provide an instance of CancellationException when calling .cancel as this is the full method signature:
{% endhint %}

协程通过抛出一个特殊的异常来处理取消操作：CancellationException 。如果您想要提供更多关于取消原因的细节，可以在调用在调用cancel()方法时传入一个CancellationException  实例，因为这是cancel()的完整方法签名：

```kotlin
fun cancel(cause: CancellationException? = null)
```

{% hint style="success" %}
If you don’t provide your own CancellationException instance, a default CancellationException will be created (full code [here](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/JobSupport.kt#L612)):
{% endhint %}
如果使用缺省调用，则会创建一个默认的CancellationException 实例（[此处](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/JobSupport.kt#L612)有完整代码）

```kotlin
public override fun cancel(cause: CancellationException?) {
    cancelInternal(cause ?: defaultCancellationException())
}
```

{% hint style="success" %}
Because CancellationException is thrown, then you will be able to use this mechanism to handle the coroutine cancellation. More about how to do this in the Handling cancellation side effects section below.
{% endhint %}
因为协程的取消会抛出CancellationException  ，所以我们可以利用这个机制来对协程的取消进行一些操作。关于如何操作的详细说明，请参见本文下面的“处理取消副作用”小节。


{% hint style="success" %}
Under the hood, the child job notifies its parent about the cancellation via the exception. The parent uses the cause of the cancellation to determine whether it needs to handle the exception. If the child was cancelled due to CancellationException, then no other action is required for the parent.
{% endhint %}
在底层，子协程的取消会通过抛出异常来通知父级，而父级根据取消的原因来确定是否需要处理异常。如果子协程是由于CancellationException  而被取消，那么父级就不需要再执行其他操作。


{% hint style="success" %}
Once you cancel a scope, you won’t be able to launch new coroutines in the cancelled scope.
{% endhint %}

{% hint style="success" %}
If you’re using the androidx KTX libraries in most cases you don’t create your own scopes and therefore you’re not responsible for cancelling them. If you’re working in the scope of a ViewModel, using viewModelScope or, if you want to launch coroutines tied to a lifecycle scope, you would use the lifecycleScope. Both viewModelScope and lifecycleScope are CoroutineScope objects that get cancelled at the right time. For example, when the ViewModel is cleared, it cancels the coroutines launched in its scope.
{% endhint %}

当使用androidx KTX库时，大多数情况我们不需要创建自己的作用域，因此我们也不负责取消它们。比如在ViewModel 中我们可以使用viewModelScope ,或者当我们想启动一个与页面生命周期相关的协程时可以使用lifecycleScope  。viewModelScope 和lifecycleScope  都是可以在正确的时间可以被自动取消的协程作用域对象。例如，当ViewModel  被清除时，也会同时取消在 viewModelScope中启动的协程。


## Why isn’t my coroutine work stopping?

{% hint style="success" %}
If we just call cancel, it doesn’t mean that the coroutine work will just stop. If you’re performing some relatively heavy computation, like reading from multiple files, there’s nothing that automatically stops your code from running.
{% endhint %}

如果我们仅调用cancel()方法，并不意味着协程的工作就会立刻停止。如果协程正在执行一些比较繁重的计算，比如从多个文件中读取数据，则不会有任何东西可以让此协程自动停止。

{% hint style="success" %}
Let’s take a more simple example and see what happens. Let’s say that we need to print “Hello” twice a second using coroutines. We’re going to let the coroutine run for a second and then cancel it. One version of the implementation can look like this:
{% endhint %}
让我们举个更简单的例子看看会发生什么。假设我们需要使用协程以每秒两次的速度打印"Hello"，我们让协程运行一秒钟然后取消它；

```kotlin
import kotlinx.coroutines.*
 
fun main(args: Array<String>) = runBlocking<Unit> {
   val startTime = System.currentTimeMillis()
    val job = launch (Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (i < 5) {
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("Hello ${i++}")
                nextPrintTime += 500L
            }
        }
    }
    delay(1000L)
    println("Cancel!")
    job.cancel()
    println("Done!")
}
```
{% hint style="success" %}
Let’s see what happens step by step. When calling launch, we’re creating a new coroutine in the active state. We’re letting the coroutine run for 1000ms. So now we see printed:
{% endhint %}
让我们一步一步看看会发生什么。当调用launch时，我们正在创建一个处于活动状态的新协程，我们让这个协程运行1000毫秒，现在我们看到了：

```
Hello 0
Hello 1
Hello 2
```
{% hint style="success" %}
Once job.cancel is called, our coroutine moves to Cancelling state. But then, we see that Hello 3 and Hello 4 are printed to the terminal. Only after the work is done, the coroutine moves to Cancelled state.
{% endhint %}
一旦调用了job.cancel()，协程会进入Cancelling  状态，但是之后我们仍然看看到了Hello3和Hello4被打印到了控制台上。只有协程在完成工作后，才会被移入 Cancelled  状态。

{% hint style="success" %}
The coroutine work doesn’t just stop when cancel is called. Rather, we need to modify our code and check if the coroutine is active periodically.
{% endhint %}
协程不会在调用job.cancel()时立即停止，所以我们需要修改代码并定期检查协程是否处于活动状态。

{% hint style="success" %}
Cancellation of coroutine code needs to be cooperative!
{% endhint %}

> ⚠️ 协程的取消是协作式的，是需要配合的。

## 使协程可被取消
{% hint style="success" %}

You need to make sure that all the coroutine work you’re implementing is cooperative with cancellation, therefore you need to check for cancellation periodically or before beginning any long running work. For example, if you’re reading multiple files from disk, before you start reading each file, check whether the coroutine was cancelled or not. Like this you avoid doing CPU intensive work when it’s not needed anymore.

我们需要确保所有协程都是与取消是协作的，因此需要定期或在任何开始长时间运行的工作之前检查是否有取消。例如，当我们从磁盘读取多个文件，在开始读取每个文件之前都应该检查协程是否已被取消，这样，当不再需要CPU时，就可以避免执行CPU密集型操作以减少消耗。

{% endhint %}
```kotlin
val job = launch {
    for(file in files) {
        // TODO check for cancellation
        readFile(file)
    }
}
```

{% hint style="success" %}
All suspend functions from kotlinx.coroutines are cancellable: withContext, delay etc. So if you’re using any of them you don’t need to check for cancellation and stop execution or throw a CancellationException. But, if you’re not using them, to make your coroutine code cooperative we have two options:
* Checking job.isActive or ensureActive()
* Let other work happen using yield()
{% endhint %}

kotlinx.coroutines中的所有挂起函数都是可以被取消的，如withContext()  , delay()  等。因此，如果我们使用它们中的任何一个，都不需要检查取消并立即停止或抛出CancellationException  。但如果不使用这些，为了使我们的协程可协作式取消，我们有两个选择：
* 使用job.isActive或ensureActive()来检查
* 使用yield()

### 检查Job的活动状态
{% hint style="success" %}
One option is in our while(i<5) to add another check for the coroutine state:
{% endhint %}

依然以上面的代码为例，第一种选择是在while(i < 5)处添加一个协程的状态检查


```kotlin
// Since we're in the launch block, we have access to job.isActive
while (i < 5 && isActive)
```
{% hint style="success" %}
This means that our work should only be executed while the coroutine is active. This also means that once we’re out of the while, if we want to do some other action, like logging if the job was cancelled, we can add a check for !isActive and do our action there.
{% endhint %}
这意味着工作应该只在协程处于活动状态时执行，同时一旦我们离开了while，如果想要执行其他操作，比如记录Job 是否被取消了，则可以添加一个检查!isActive。 协程库提供了另一个很有用的方法 —ensureActive()，它的实现是：
{% hint style="success" %}
The Coroutines library provides another helpful method - ensureActive(). Its implementation is:
{% endhint %}
因为这个方法会在Job 处于不活跃时立即抛出异常，所以我们可以将其作为while循环中的第一个操作：

```kotlin
fun Job.ensureActive(): Unit {
    if (!isActive) {
         throw getCancellationException()
    }
}
```
{% hint style="success" %}
Because this method instantaneously throws if the job is not active, we can make this the first thing we do in our while loop:
{% endhint %}

因为这个方法会在Job 处于不活跃时立即抛出异常，所以我们可以将其作为while循环中的第一个操作：
```kotlin
while (i < 5) {
    ensureActive()
    …
}
```
通过使用ensureActive()方法，我们可以避免自己实现isActive所需的if语句从而减少需要编写的样板代码，但是也失去了执行其他操作（比如日志记录）的灵活性。

### 使用yield()

{% hint style="success" %}
If the work you’re doing is 1) CPU heavy, 2) may exhaust the thread pool and 3) you want to allow the thread to do other work without having to add more threads to the pool, then use yield(). The first operation done by yield will be checking for completion and exit the coroutine by throwing CancellationException if the job is already completed. yield can be the first function called in the periodic check, like ensureActive() mentioned above.
{% endhint %}

如果我们想要执行的操作是 1）占用大量CPU资源，2）可能耗尽线程池，3）以及希望允许线程执行其他工作而不必向池中添加更多线程，应使用。yield()的第一个操作就是检查Job  是否完成，如果Job  完成，则通过抛出CancellationException  来退出协程yield()可以作为定期检查中的第一个函数，就像上文中的ensureActive()


## Job.join vs Deferred.await cancellation

{% hint style="success" %}
There are two ways to wait for a result from a coroutine: jobs returned from launch can call join and Deferred (a type of Job) returned from async can be await’d.
{% endhint %}

有两种方式可以等待协程的结果：从launch  返回的Job  实例可以调用join  方法，从async  返回的Deferred  （Job  的子类）可以调用await  方法。


{% hint style="success" %}
Job.join suspends a coroutine until the work is completed. Together with job.cancel it behaves as you’d expect:
* If you’re calling job.cancel then job.join, the coroutine will suspend until the job is completed.
* Calling job.cancel after job.join has no effect, as the job is already completed.
{% endhint %}
Job.join()会挂起一个协程直到协程的工作完成。和Job.cancel()一起使用会根据我们的调用顺序产生结果：

* 如果先调用和Job.cancel()然后调用Job.join()，协程将挂起直到Job 完成。
* 如果先调用Job.join()然后调用和Job.cancel()，将不会产生任何效果，因为协程已经完成了。

{% hint style="success" %}
You use a Deferred when you are interested in the result of the coroutine. This result is returned by Deferred.await when the coroutine is completed. Deferred is a type of Job, and it can also be cancelled.
{% endhint %}
当我们对协程的结果更感兴趣时，可以使用Deferred  。当协程完成时，结果会通过Deferred.await()返回。Deferred  是Job  的子类，所以它也是可被取消的。


{% hint style="success" %}
Calling await on a deferred that was cancelled throws a JobCancellationException.
{% endhint %}
对已经被取消的Deferred  调用await()会抛出JobCancellationException  异常。

```kotlin
val deferred = async { … }
deferred.cancel()
val result = deferred.await() // throws JobCancellationException!
```
{% hint style="success" %}
Here’s why we get the exception: the role of await is to suspend the coroutine until the result is computed; since the coroutine is cancelled, the result cannot be computed. Therefore, calling await after cancel leads to JobCancellationException: Job was cancelled.
{% endhint %}
因为await  的作用是挂起协程直到得到结果，由于协程已经被取消，因此无法再计算结果。因此，取消后再调用await  会导致JobCancellationException: Job was cancelled。

{% hint style="success" %}
On the other hand, if you’re calling deferred.cancel after deferred.await nothing happens, as the coroutine is already completed.
{% endhint %}
另一方面，如果在deferred.await()之后调用deferred.cancel()不会有任何效果，因为协程已经完成了。

## 处理取消的副作用
{% hint style="success" %}
Let’s say that you want to execute a specific action when a coroutine is cancelled: closing any resources you might be using, logging the cancellation or some other cleanup code you want to execute. There are several ways we can do this:
{% endhint %}
假设我们想在协程被取消时执行某个特定的操作比如关闭可能正在使用的任何资源，记录取消或执行其他清理代码，有几种方法可以做到这一点：


### 使用isActive检查

{% hint style="success" %}
If you’re periodically checking for isActive, then once you’re out of the while loop, you can clean up the resources. Our code above could be updated to:
{% endhint %}
如果我们定期检查了isActive 的状态，那么一旦退出while循环，我们就可以做一些清理资源的工作，上面的代码可以更新为：

```kotlin
while (i < 5 && isActive) {
    // print a message twice a second
    if (…) {
        println(“Hello ${i++}”)
        nextPrintTime += 500L
    }
}
// the coroutine work is completed so we can cleanup
println(“Clean up!”)
```
{% hint style="success" %}
See it in action here.
So now, when the coroutine is no longer active, the while will break and we can do our cleanup.
{% endhint %}
可以在这里看看运行效果。
所以现在，当协程不再活跃时，while循环会中断，我们可以进行清理一些资源。

### Try catch finally
{% hint style="success" %}
Since a CancellationException is thrown when a coroutine is cancelled, then we can wrap our suspending work in try/catch and in the finally block, we can implement our clean up work.
{% endhint %}
由于协程被取消时会抛出一个CancellationException ，因此可以将协程包装到try/catch 和finally 块中，从而执行一些清理工作：
```kotlin
val job = launch {
   try {
      work()
   } catch (e: CancellationException){
      println(“Work cancelled!”)
    } finally {
      println(“Clean up!”)
    }
}
delay(1000L)
println(“Cancel!”)
job.cancel()
println(“Done!”)
```

{% hint style="success" %}
But, if the cleanup work we need to execute is suspending, the code above won’t work anymore, as once the coroutine is in Cancelling state, it can’t suspend anymore. See the full code here.
{% endhint %}
但是，如果我们需要执行的清理工作是需要挂起的，则上面的代码不再起作用，因为一旦协程处于Cancelling 的状态，那么它就不能再次挂起。

{% hint style="success" %}
A coroutine in the cancelling state is not able to suspend!
{% endhint %}
> ⚠️ 处于取消状态的协程不能再次挂起！

{% hint style="success" %}
To be able to call suspend functions when a coroutine is cancelled, we will need to switch the cleanup work we need to do in a NonCancellable CoroutineContext. This will allow the code to suspend and will keep the coroutine in the Cancelling state until the work is done.
{% endhint %}
为了能够再协程被取消时调用挂起函数，我们需要在一个不可取消的协程上下文中切换清理工作，这将允许代码挂起，并将协程保持在Cancelling 状态直至完成工作：
```kotlin
val job = launch {
   try {
      work()
   } catch (e: CancellationException){
      println(“Work cancelled!”)
    } finally {
      withContext(NonCancellable){
         delay(1000L) // or some other suspend fun 
         println(“Cleanup done!”)
      }
    }
}
delay(1000L)
println(“Cancel!”)
job.cancel()
println(“Done!”)
```
{% hint style="success" %}
Check out how this works in practice here.
{% endhint %}
可以在此处实践上述代码。

## suspendCancellableCoroutine and invokeOnCancellation
{% hint style="success" %}
If you converted callbacks to coroutines by using the suspendCoroutine method, then prefer using suspendCancellableCoroutine instead. The work to be done on cancellation can be implemented using continuation.invokeOnCancellation:
{% endhint %}
如果使用suspendCoroutine 方法将回调改成协程，那么最好使用suspendCancellableCoroutine 。可以使用continuation.invokeOnCancellation来完成取消工作：


```kotlin
suspend fun work() {
   return suspendCancellableCoroutine { continuation ->
       continuation.invokeOnCancellation { 
          // do cleanup
       }
   // rest of the implementation
}
```
{% hint style="success" %}
To realise the benefits of structured concurrency and ensure that we’re not doing unnecessary work you need to make sure that you’re also making your code cancellable.
{% endhint %}
为了安全的享用结构化并发的好处并不做不必要的工作，我们需要确保我们协程是可取消的。
{% hint style="success" %}
Use the CoroutineScopes defined in Jetpack: viewModelScope or lifecycleScope that cancels their work when their scope completes. If you’re creating your own CoroutineScope, make sure you’re tying it to a job and calling cancel when needed.
{% endhint %}
使用在JetPack中定义的CoroutineScopes  （viewModelScope   或 lifecycleScope  ）可确保当作用域结束时内部的协程也会取消。如果我们要创建自己的CoroutineScopes  ，应将它绑定到一个Job  并在需要时取消。

{% hint style="success" %}
The cancellation of coroutine code needs to be cooperative so make sure you update your code to check for cancellation to be lazy and avoid doing more work than necessary.
{% endhint %}
协程的取消时协作式的，所以在使用协程时要确保它的取消是惰性的以避免执行不必要的操作。

{% hint style="success" %}
Find out more about patterns for work that shouldn’t be cancelled from this post:
{% endhint %}

{% embed url="https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad" %}