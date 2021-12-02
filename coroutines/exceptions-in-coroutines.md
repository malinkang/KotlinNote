We, developers, usually spend a lot of time polishing the happy path of our app. However, it’s equally important to provide a proper user experience whenever things don’t go as expected. On one hand, seeing an application crash is a bad experience for the user; on the other hand, showing the right message to the user when an action didn’t succeed is indispensable.
Handling exceptions properly has a huge impact on how users perceive your application. In this article, we’ll explain how exceptions are propagated in coroutines and how you can always be in control, including the different ways to handle them.
If you prefer video, check out this talk from KotlinConf’19 by 
Florina Muntenescu
 and I:

⚠️ In order to follow the rest of the article without any problems, reading and understanding Part 1 of the series is required.
Coroutines: First things first
Cancellation and Exceptions in Coroutines (Part 1)
medium.com

## 一个协程突然失败了！现在该怎么办？😱

{% hint style="success" %}
When a coroutine fails with an exception, it will propagate said exception up to its parent! Then, the parent will 1) cancel the rest of its children, 2) cancel itself and 3) propagate the exception up to its parent.
{% endhint %}

当协程因出现异常失败时，它会将异常传播到它的父级，然后，父级将进行如下三步：1）取消其余的子协程，2）取消自身，3）将异常在传播给它的父级。

{% hint style="success" %}
The exception will reach the root of the hierarchy and all the coroutines that the CoroutineScope started will get cancelled too.
{% endhint %}
异常将最终到达当前层次结构的根，在当前协程作用域启动的所有协程都将被取消。

{% hint style="success" %}
An exception in a coroutine will be propagated throughout the coroutines hierarchy
While propagating an exception can make sense in some cases, there are other cases when that’s undesirable. Imagine a UI-related CoroutineScope that processes user interactions. If a child coroutine throws an exception, the UI scope will be cancelled and the whole UI component will become unresponsive as a cancelled scope cannot start more coroutines.
{% endhint %}

虽然在某些情况下传播异常是有意义的，但是大多数情况下这样做是不可取的。设想一个处理用户交互的与UI相关的协程作用域，如果一个子协程抛出了异常，UI的作用域将被取消，整个UI组件将变得无响应，因为已经取消的协程作用域无法再此启动协程。

{% hint style="success" %}
What if you don’t want that behavior? Alternatively, you can use a different implementation of Job, namely SupervisorJob, in the CoroutineContext of the CoroutineScope that creates these coroutines.
{% endhint %}

如果这不是我们的预期行为我们该怎么办呢？作为一种选择，我们可以在当前协程作用域的上下文中使用Job 的另一种实现：SupervisorJob 。

### SupervisorJob to the rescue

{% hint style="success" %}
With a SupervisorJob, the failure of a child doesn’t affect other children. A SupervisorJob won’t cancel itself or the rest of its children. Moreover, SupervisorJob won’t propagate the exception either, and will let the child coroutine handle it.
{% endhint %}
使用SupervisorJob ，子协程的失败不会影响到其他子协程。SupervisorJob 不会取消自身或它的其他子协程，而且SupervisorJob 不会传播异常而是让它的协程处理。
    
{% hint style="success" %}
You can create a CoroutineScope like this val uiScope = CoroutineScope(SupervisorJob()) to not propagate cancellation when a coroutine fails as this image depicts:
{% endhint %}
你可以像这样val uiScope = CoroutineScope(SupervisorJob()) 创建一个SupervisorJob ，以便在协程因异常失败时不会传播，如下图所示；



A SupervisorJob won’t cancel itself or the rest of its children

{% hint style="success" %}
If the exception is not handled and the CoroutineContext doesn’t have a CoroutineExceptionHandler (as we’ll see later), it will reach the default thread’s ExceptionHandler. In the JVM, the exception will be logged to console; and in Android, it will make your app crash regardless of the Dispatcher this happens on.
{% endhint %}
如果异常没有被处理，并且协程上下文（CoroutineContext）中没有CoroutineExceptionHandler （我们将在后面讲它），那么它将到达默认线程的ExceptionHandler 。在JVM中，异常将被记录到控制台；在Android中，不论它发生在哪个调度器中都会使App崩溃。


💥 Uncaught exceptions will always be thrown regardless of the kind of Job you use
The same behavior applies to the scope builders coroutineScope and supervisorScope. These will create a sub-scope (with a Job or a SupervisorJob accordingly as a parent) with which you can logically group coroutines (e.g. if you want to do parallel computations or you want them to be or not be affected by each other).
Warning: A SupervisorJob only works as described when it’s part of a scope: either created using supervisorScope or CoroutineScope(SupervisorJob()).


### Job还是SupervisorJob? 🤔

{% hint style="success" %}
When should you use a Job or a SupervisorJob? Use a SupervisorJob or supervisorScope when you don’t want a failure to cancel the parent and siblings.
Some examples:
{% endhint %}
什么时候应该使用Job 或SupervisorJob ？当我们不想因异常取消父级或同级协程时，使用SupervisorJob 或supervisorScope{...} 。
一些例子：

```kotlin
// Scope handling coroutines for a particular layer of my app
val scope = CoroutineScope(SupervisorJob())
scope.launch {
    // Child 1
}
scope.launch {
    // Child 2
}
```
{% hint style="success" %}
In this case, if child#1 fails, neither scope nor child#2 will be cancelled.
Another example:
{% endhint %}
在这种情况下，如果Child 1失败，scope和Child 2都不会被取消。
```kotlin
// Scope handling coroutines for a particular layer of my app
val scope = CoroutineScope(Job())
scope.launch {
    supervisorScope {
        launch {
            // Child 1
        }
        launch {
            // Child 2
        }
    }
}
```
{% hint style="success" %}
In this case, as supervisorScope creates a sub-scope with a SupervisorJob, if child#1 fails, child#2 will not be cancelled. If instead you use a coroutineScope in the implementation, the failure will get propagated and will end up cancelling scope too.
{% endhint %}
在这种情况下，由于supervisorScope{...} 创建了一个带有SupervisorJob 的作用域，如果Child 1失败，Child 2不会被取消。如果使用coroutineScope{...} ，则会传播异常并最终取消scope 。

### 注意测试！谁是我的父级？ 🎯
{% hint style="success" %}
Given the following snippet of code, can you identify what kind of Job child#1 has as a parent?
{% endhint %}
```kotlin
val scope = CoroutineScope(Job())
scope.launch(SupervisorJob()) {
    // new coroutine -> can suspend
   launch {
        // Child 1
    }
    launch {
        // Child 2
    }
}
```
{% hint style="success" %}
child#1’s parentJob is of type Job! Hope you got it right! Even though at first impression, you might’ve thought that it can be a SupervisorJob, it is not because a new coroutine always gets assigned a new Job() which in this case overrides the SupervisorJob. SupervisorJob is the parent of the coroutine created with scope.launch; so literally, SupervisorJob does nothing in that code!
{% endhint %}

Child 1的父Job是Job ！希望你做对了，乍一看你可能认为它的父级是SupervisorJob 。在这种情况下，并不是因为新协程总会被分配一个新Job 而覆盖了SupervisorJob ，该SupervisorJob 是使用scope.launch 创建的协程的父级，还记得上面的警告吗？所以字面上来看SupervisorJob 在该代码中什么也没有做。
{% hint style="success" %}
The parent of child#1 and child#2 is of type Job, not SupervisorJob
Therefore, if either child#1 or child#2 fails, the failure will reach scope and all work started by that scope will be cancelled.
{% endhint %}
{% hint style="success" %}
Remember that a SupervisorJob only works as described when it’s part of a scope: either created using supervisorScope or CoroutineScope(SupervisorJob()). Passing a SupervisorJob as a parameter of a coroutine builder will not have the desired effect you would’ve thought for cancellation.
{% endhint %}
请记住，SupervisorJob 只有在supervisorScope{...} 或CoroutineScope(SupervisorJob()) 创建的作用域中才会有效，将SupervisorJob 作为协程构建器的参数时将无法产生我们的预期行为。
{% hint style="success" %}
Regarding exceptions, if any child throws an exception, that SupervisorJob won’t propagate the exception up in the hierarchy and will let its coroutine handle it.
{% endhint %}
### 底层原理
{% hint style="success" %}
If you’re curious about how Job works under the hood, check out the implementation of the functions childCancelled and notifyCancelling in the JobSupport.kt file.
{% endhint %}
如果你对Job 的工作原理感兴趣，请查看JobSupport.kt文件中childCancelled 和notifyCancelling函数的实现。
{% hint style="success" %}
In the SupervisorJob implementation, the childCancelled method just returns false, meaning that it doesn’t propagate cancellation but it doesn’t handle the exception either.
{% endhint %}
在SupervisorJob 的实现中，childCancelled() 方法只返回false，这意味着它不会传播取消，但也不会处理异常。

## 处理异常 👩‍🚒
{% hint style="success" %}
Coroutines use the regular Kotlin syntax for handling exceptions: try/catch or built-in helper functions like runCatching (which uses try/catch internally).
{% endhint %}
协程使用常规的Koltin语法来处理异常：try/catch 或内置的辅助函数如runCatching (它在内部也使用try/catch )
{% hint style="success" %}
We said before that uncaught exceptions will always be thrown. However, different coroutines builders treat exceptions in different ways.
{% endhint %}
我们之前说过，未捕获的异常总是会被抛出。然而不同的协程构建器会以不同的方式来处理异常。

### Launch

With launch, exceptions will be thrown as soon as they happen. Therefore, you can wrap the code that can throw exceptions inside a try/catch, like in this example:
scope.launch {
    try {
        codeThatCanThrowExceptions()
    } catch(e: Exception) {
        // Handle exception
    }
}
With launch, exceptions will be thrown as soon as they happen
### Async

When async is used as a root coroutine (coroutines that are a direct child of a CoroutineScope instance or supervisorScope), exceptions are not thrown automatically, instead, they’re thrown when you call .await().
To handle exceptions thrown in async whenever it’s a root coroutine, you can wrap the .await() call inside a try/catch:
```kotlin
supervisorScope {
    val deferred = async {
        codeThatCanThrowExceptions()
    }
    try {
        deferred.await()
    } catch(e: Exception) {
        // Handle exception thrown in async
    }
}
```
{% hint style="success" %}
In this case, notice that calling async will never throw the exception, that’s why it’s not necessary to wrap it as well. await will throw the exception that happened inside the async coroutine.
{% endhint %}
请注意，在这种情况下，调用async将永远不会抛出异常，这就是为什么不用将它也封装到try/catch 代码块里的原因：.await() 会抛出在async内部发生的异常。

{% hint style="success" %}
When async is used as a root coroutine, exceptions are thrown when you call .await
{% endhint %}

> 当async用作根协程时，会在调用.await()处抛出异常。

{% hint style="success" %}
Also, notice that we’re using a supervisorScope to call async and await. As we said before, a SupervisorJob lets the coroutine handle the exception; as opposed to Job that will automatically propagate it up in the hierarchy so the catch block won’t be called:
{% endhint %}
此外，在由其他协程构建器创建的协程发生的异常也会始终被传播，例如：
```kotlin
coroutineScope {
    try {
        val deferred = async {
            codeThatCanThrowExceptions()
        }
        deferred.await()
    } catch(e: Exception) {
        // Exception thrown in async WILL NOT be caught here 
        // but propagated up to the scope
    }
}
```
Furthermore, exceptions that happen in coroutines created by other coroutines will always be propagated regardless of the coroutine builder. For example:
此外，在由其他协程构建器创建的协程发生的异常也会始终被传播，例如：

```kotlin
val scope = CoroutineScope(Job())
scope.launch {
    async {
        // If async throws, launch throws without calling .await()
    }
}
```

{% hint style="success" %}
In this case, if async throws an exception, it will get thrown as soon as it happens because the coroutine that is the direct child of the scope is launch. The reason is that async (with a Job in its CoroutineContext) will automatically propagate the exception up to its parent (launch) that will throw the exception.
{% endhint %}

在这个例子中，如果async会抛出一个异常，将在异常发生时立即被抛出，因为scope的直接子协程由launch构建，而async的协程上下文中的Job实例为Job ，所以async构建的协程发生异常时将自动向上级传播。

⚠️ Exceptions thrown in a coroutineScope builder or in coroutines created by other coroutines won’t be caught in a try/catch!
In the SupervisorJob section, we mention the existence of CoroutineExceptionHandler. Let’s dive into it!


### CoroutineExceptionHandler

The CoroutineExceptionHandler is an optional element of a CoroutineContext allowing you to handle uncaught exceptions.
Here’s how you can define a CoroutineExceptionHandler, whenever an exception is caught, you have information about the CoroutineContext where the exception happened and the exception itself:
val handler = CoroutineExceptionHandler {
    context, exception -> println("Caught $exception")
}

{% hint style="success" %}
Exceptions will be caught if these requirements are met:
When ⏰: The exception is thrown by a coroutine that automatically throws exceptions (works with launch, not with async).
Where 🌍: If it’s in the CoroutineContext of a CoroutineScope or a root coroutine (direct child of CoroutineScope or a supervisorScope).
{% endhint %}
注意当满足以下要求，才会捕获异常：

When ⏰   :发生异常的协程会自动抛出异常，适用于使用launch构建的协程
Where 🌍 : 发生异常的位置为协程作用域（CoroutineScope）的上下文（CoroutineContext）中或者根协程（协程作用域直接管理的协程）

{% hint style="success" %}
Let’s see some examples using the CoroutineExceptionHandler defined above. In the following example, the exception will be caught by the handler:
{% endhint %}

让我们来看一些使用上面定义的CoroutineExceptionHandler 的例子，在下面的这个例子中，异常将被CoroutineExceptionHandler 捕获：
```kotlin
val scope = CoroutineScope(Job())
scope.launch(handler) {
    launch {
        throw Exception("Failed coroutine")
    }
}
```
{% hint style="success" %}
In this other case in which the handler is installed in a inner coroutine, it won’t be caught:
{% endhint %}
在另一种情况下，CoroutineExceptionHandler 安装在一个内部协程时，它不会再捕获异常：


```kotlin
val scope = CoroutineScope(Job())
scope.launch {
    launch(handler) {
        throw Exception("Failed coroutine")
    }
}
```
{% hint style="success" %}
The exception isn’t caught because the handler is not installed in the right CoroutineContext. The inner launch will propagate the exception up to the parent as soon as it happens, since the parent doesn’t know anything about the handler, the exception will be thrown.
{% endhint %}
因为没有在正确的CoroutineContext 使用CoroutineExceptionHandler ，内部协程会在异常发生时立即向上级抛出异常，而父级不知道任何关于CoroutineExceptionHandler 的信息，所以异常将被抛出。
{% hint style="success" %}
Dealing with exceptions gracefully in your application is important to have a good user experience, even when things don’t go as expected.
{% endhint %}
在应用程序中优雅的处理异常对于获得良好的用户体验非常重要，即使事情并没有按照预期进行。

{% hint style="success" %}
Remember to use SupervisorJob when you want to avoid propagating cancellation when an exception happens, and Job otherwise.
{% endhint %}
请记住，当我们希望避免发生异常时异常被传播使用SupervisorJob ，否则用Job 。
{% hint style="success" %}
Uncaught exceptions will be propagated, catch them to provide a great UX!
{% endhint %}
未捕获的异常将被传播，捕获它们以提供更好的用户体验！

