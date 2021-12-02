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

## Calling cancel

{% hint style="success" %}
When launching multiple coroutines, it can be a pain to keep track of them or cancel each individually. Rather, we can rely on cancelling the entire scope coroutines are launched into as this will cancel all of the child coroutines created:
{% endhint %}

```kotlin
// assume we have a scope defined for this layer of the app
val job1 = scope.launch { … }
val job2 = scope.launch { … }
scope.cancel()
```

{% hint style="success" %}
Cancelling the scope cancels its children
{% endhint %}

{% hint style="success" %}
Sometimes you might need to cancel only one coroutine, maybe as a reaction to a user input. Calling job1.cancel ensures that only that specific coroutine gets cancelled and all the other siblings are not affected:
{% endhint %}

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

{% hint style="success" %}
Coroutines handle cancellation by throwing a special exception: CancellationException. If you want to provide more details on the cancellation reason you can provide an instance of CancellationException when calling .cancel as this is the full method signature:
{% endhint %}

```kotlin
fun cancel(cause: CancellationException? = null)
```

{% hint style="success" %}
If you don’t provide your own CancellationException instance, a default CancellationException will be created (full code here):
{% endhint %}

```kotlin
public override fun cancel(cause: CancellationException?) {
    cancelInternal(cause ?: defaultCancellationException())
}
```
{% hint style="success" %}
Because CancellationException is thrown, then you will be able to use this mechanism to handle the coroutine cancellation. More about how to do this in the Handling cancellation side effects section below.
{% endhint %}

{% hint style="success" %}
Under the hood, the child job notifies its parent about the cancellation via the exception. The parent uses the cause of the cancellation to determine whether it needs to handle the exception. If the child was cancelled due to CancellationException, then no other action is required for the parent.
{% endhint %}

{% hint style="success" %}
Once you cancel a scope, you won’t be able to launch new coroutines in the cancelled scope.
{% endhint %}

{% hint style="success" %}
If you’re using the androidx KTX libraries in most cases you don’t create your own scopes and therefore you’re not responsible for cancelling them. If you’re working in the scope of a ViewModel, using viewModelScope or, if you want to launch coroutines tied to a lifecycle scope, you would use the lifecycleScope. Both viewModelScope and lifecycleScope are CoroutineScope objects that get cancelled at the right time. For example, when the ViewModel is cleared, it cancels the coroutines launched in its scope.
{% endhint %}

## Why isn’t my coroutine work stopping?
{% hint style="success" %}
If we just call cancel, it doesn’t mean that the coroutine work will just stop. If you’re performing some relatively heavy computation, like reading from multiple files, there’s nothing that automatically stops your code from running.
{% endhint %}

{% hint style="success" %}
Let’s take a more simple example and see what happens. Let’s say that we need to print “Hello” twice a second using coroutines. We’re going to let the coroutine run for a second and then cancel it. One version of the implementation can look like this:
{% endhint %}

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

```
Hello 0
Hello 1
Hello 2
```
{% hint style="success" %}
Once job.cancel is called, our coroutine moves to Cancelling state. But then, we see that Hello 3 and Hello 4 are printed to the terminal. Only after the work is done, the coroutine moves to Cancelled state.
{% endhint %}
{% hint style="success" %}
The coroutine work doesn’t just stop when cancel is called. Rather, we need to modify our code and check if the coroutine is active periodically.
{% endhint %}
{% hint style="success" %}
Cancellation of coroutine code needs to be cooperative!
{% endhint %}

## Making your coroutine work cancellable
{% hint style="success" %}
You need to make sure that all the coroutine work you’re implementing is cooperative with cancellation, therefore you need to check for cancellation periodically or before beginning any long running work. For example, if you’re reading multiple files from disk, before you start reading each file, check whether the coroutine was cancelled or not. Like this you avoid doing CPU intensive work when it’s not needed anymore.
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

## Checking for job’s active state
{% hint style="success" %}
One option is in our while(i<5) to add another check for the coroutine state:
{% endhint %}

```kotlin
// Since we're in the launch block, we have access to job.isActive
while (i < 5 && isActive)
```
{% hint style="success" %}
This means that our work should only be executed while the coroutine is active. This also means that once we’re out of the while, if we want to do some other action, like logging if the job was cancelled, we can add a check for !isActive and do our action there.
{% endhint %}
{% hint style="success" %}
The Coroutines library provides another helpful method - ensureActive(). Its implementation is:
{% endhint %}

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
```kotlin
while (i < 5) {
    ensureActive()
    …
}
```

## Let other work happen using yield()

{% hint style="success" %}
If the work you’re doing is 1) CPU heavy, 2) may exhaust the thread pool and 3) you want to allow the thread to do other work without having to add more threads to the pool, then use yield(). The first operation done by yield will be checking for completion and exit the coroutine by throwing CancellationException if the job is already completed. yield can be the first function called in the periodic check, like ensureActive() mentioned above.
{% endhint %}

## Job.join vs Deferred.await cancellation

{% hint style="success" %}
There are two ways to wait for a result from a coroutine: jobs returned from launch can call join and Deferred (a type of Job) returned from async can be await’d.
{% endhint %}

{% hint style="success" %}
Job.join suspends a coroutine until the work is completed. Together with job.cancel it behaves as you’d expect:
* If you’re calling job.cancel then job.join, the coroutine will suspend until the job is completed.
* Calling job.cancel after job.join has no effect, as the job is already completed.
{% endhint %}
{% hint style="success" %}
You use a Deferred when you are interested in the result of the coroutine. This result is returned by Deferred.await when the coroutine is completed. Deferred is a type of Job, and it can also be cancelled.
{% endhint %}
{% hint style="success" %}
Calling await on a deferred that was cancelled throws a JobCancellationException.
{% endhint %}
```kotlin
val deferred = async { … }
deferred.cancel()
val result = deferred.await() // throws JobCancellationException!
```
{% hint style="success" %}
Here’s why we get the exception: the role of await is to suspend the coroutine until the result is computed; since the coroutine is cancelled, the result cannot be computed. Therefore, calling await after cancel leads to JobCancellationException: Job was cancelled.
{% endhint %}
{% hint style="success" %}
On the other hand, if you’re calling deferred.cancel after deferred.await nothing happens, as the coroutine is already completed.
{% endhint %}
## Handling cancellation side effects
{% hint style="success" %}
Let’s say that you want to execute a specific action when a coroutine is cancelled: closing any resources you might be using, logging the cancellation or some other cleanup code you want to execute. There are several ways we can do this:
{% endhint %}
### Check for !isActive
{% hint style="success" %}
If you’re periodically checking for isActive, then once you’re out of the while loop, you can clean up the resources. Our code above could be updated to:
{% endhint %}
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
## Try catch finally
{% hint style="success" %}
Since a CancellationException is thrown when a coroutine is cancelled, then we can wrap our suspending work in try/catch and in the finally block, we can implement our clean up work.
{% endhint %}
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
{% hint style="success" %}
A coroutine in the cancelling state is not able to suspend!
{% hint style="success" %}
To be able to call suspend functions when a coroutine is cancelled, we will need to switch the cleanup work we need to do in a NonCancellable CoroutineContext. This will allow the code to suspend and will keep the coroutine in the Cancelling state until the work is done.
{% endhint %}
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
## suspendCancellableCoroutine and invokeOnCancellation
{% hint style="success" %}
If you converted callbacks to coroutines by using the suspendCoroutine method, then prefer using suspendCancellableCoroutine instead. The work to be done on cancellation can be implemented using continuation.invokeOnCancellation:
{% endhint %}

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
{% hint style="success" %}
Use the CoroutineScopes defined in Jetpack: viewModelScope or lifecycleScope that cancels their work when their scope completes. If you’re creating your own CoroutineScope, make sure you’re tying it to a job and calling cancel when needed.
{% endhint %}
{% hint style="success" %}
The cancellation of coroutine code needs to be cooperative so make sure you update your code to check for cancellation to be lazy and avoid doing more work than necessary.
{% endhint %}
{% hint style="success" %}
Find out more about patterns for work that shouldn’t be cancelled from this post:
{% endhint %}

{% embed url="https://medium.com/androiddevelopers/coroutines-patterns-for-work-that-shouldnt-be-cancelled-e26c40f142ad" %}