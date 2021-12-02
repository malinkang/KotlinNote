# Kotlin协程的取消

[原文](https://medium.com/androiddevelopers/cancellation-in-coroutines-aa6b90163629)

{% hint style="info" %}
In development, as in life, we know it’s important to avoid doing more work than needed as it can waste memory and energy. This principle applies to coroutines as well. You need to make sure that you control the life of the coroutine and cancel it when it’s no longer needed — this is what structured concurrency represents. Read on to find out the ins and outs of coroutine cancellation.
{% endhint %}

{% hint style="info" %}
If you prefer to see a video on this check out the talk&#x20;

[Manuel Vivo](https://medium.com/u/3b5622dd813c?source=post\_page-----aa6b90163629-----------------------------------) and I gave at KotlinConf’19 on coroutines cancellation and exceptions:
{% endhint %}

作为开发者，我们通常会花费大量的时间来完善我们的应用程序。然而，当事情不尽如人意时，提供合适的用户体验也同样重要。一方面，看到应用程序崩溃对用户来说是一种很糟糕的体验；另一方面，当一个操作失败时，向用户显示正确的提示信息也是必不可少的。 正确的处理异常对用户如何看待我们的App尤为重要。在本文中，我们将解释异常时如何在协程中传播的，以及如何始终进行控制，包括处理异常的不同方法。

