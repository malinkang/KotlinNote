---
title: 《Kotlin实战》读书笔记 第8章 Lambda作为形参和返回值
date: 2018-09-12 12:26:55
tags: ["Kotlin"]
---

## 8.1 声明高阶函数

高阶函数就是以另一个函数作为参数或者返回值的函数。

### 8.1.1 函数类型

![Kotlin&#x4E2D;&#x51FD;&#x6570;&#x7C7B;&#x578B;&#x8BED;&#x6CD5;](../.gitbook/assets/image%20%286%29.png)

```kotlin
    val sum = { x: Int, y: Int -> x + y }
    val action = { println(42)}
    run {
        println(sum(1,2)) //3
    }
    run{
        action() //42
    }
```

```kotlin
val sum: (Int, Int) -> Int = { x, y -> x + y } // 有两个Int型参数和Int型返回值的函数
val action: () -> Unit = { println(42) } //没有参数和返回值的函数
```

### 8.1.2 调用作为参数的函数

```kotlin
fun twoAndThree(operation: (Int, Int) -> Int) {
    val result = operation(2, 3)
    println("The result is $result")
}
```

```kotlin
twoAndThree { a, b -> a + b } //The result is 5
twoAndThree { a, b -> a * b } //The result is 6
```

### 8.1.3 在Java中使用函数类

```kotlin
LambdaTestKt.twoAndThree((a, b) -> a + b); //The result is 5
LambdaTestKt.twoAndThree((a, b) -> a * b); //The result is 6
```

### 8.1.4 函数类型的参数默认值和null值

```kotlin
fun <T> Collection<T>.joinToString(separator: String = "",
                                   prefix: String = "",
                                   postfix: String,
                                   transform: (T) -> String = { it.toString() }): String {
    val result = StringBuilder(prefix)
    for ((index, element) in withIndex()) {
        if (index > 0) result.append(separator)
        result.append(transform(element))
    }
    result.append(postfix)
    return result.toString()
}
```

```kotlin
val letters = listOf("Alpha", "Beta")
println(letters.joinToString()) //Alpha, Beta
println(letters.joinToString(transform = String::toLowerCase)) //alpha, beta
println(letters.joinToString(separator = "! ", postfix = "! ", transform = String::toUpperCase)) //ALPHA! BETA!
```


```kotlin
fun <T> Collection<T>.joinToString(separator: String = "",
                                   prefix: String = "",
                                   postfix: String,
                                   transform: ((T) -> String )?): String {
    val result = StringBuilder(prefix)
    for ((index, element) in withIndex()) {
        if (index > 0) result.append(separator)    
        result.append(transform?.invoke(element))
    }
    result.append(postfix)
    return result.toString()

}
```


### 8.1.5 返回函数的函数

```kotlin
enum class Delivery {STANDARD, EXPEDITED }

class Order(val itemCount: Int)

fun getShippingCostCalculator(delivery: Delivery): (Order) -> Double {
    if (delivery == Delivery.EXPEDITED) {
        return { order -> 6 + 2.1 * order.itemCount }
    }
    return { order -> 1.2 * order.itemCount }
}
```

```kotlin
val calculator = getShippingCostCalculator(Delivery.EXPEDITED)
println("Shipping costs ${calculator(Order(3))}") //Shipping costs 12.3
```

```kotlin
class ContactListFilters {
    var prefix: String = ""
    var onlyWithPhoneNumber: Boolean = false
    fun getPredicate(): (Person) -> Boolean {
        val startsWithPrefix = { p: Person ->
            p.firstName.startsWith(prefix) || p.lastName.startsWith(prefix)
        }
        if (!onlyWithPhoneNumber) {
            return startsWithPrefix
        }
        return { startsWithPrefix(it) && it.phoneNumber != null }
    }
}

data class Person(val firstName: String, val lastName: String, val phoneNumber: String?)
```

```kotlin
    val contacts = listOf(Person("Dmitry", "Jemerov", "123-4567"),
            Person("Svetlana", "Isakova", null))
    val contactListFilters = ContactListFilters()
    with(contactListFilters) {
        prefix = "Dm"
        onlyWithPhoneNumber = true
    }
    println(contacts.filter(contactListFilters.getPredicate()))
    //[Person(firstName=Dmitry, lastName=Jemerov, phoneNumber=123-4567)]
```

### 8.1.6 通过lambda去除重复代码

```kotlin
data class SiteVisit(val path: String, val duration: Double, val os: OS)

enum class OS {WINDOWS, LINUX, MAC, IOS, ANDROID }
```

```kotlin
    val log = listOf(
            SiteVisit("/",34.0,OS.WINDOWS),
            SiteVisit("/",22.0,OS.MAC),
            SiteVisit("/login",12.0,OS.WINDOWS),
            SiteVisit("/signup",8.0,OS.IOS),
            SiteVisit("/",16.3,OS.ANDROID)
    )
    val averageWindowsDuration = log
            .filter { it.os==OS.WINDOWS }
            .map (SiteVisit::duration)
            .average()
    println(averageWindowsDuration) //23.0
```


```text
fun List<SiteVisit>.averageDurationFor(os:OS) = filter { it.os==os }.map (SiteVisit::duration).average()
```


```kotlin
    val log = listOf(
            SiteVisit("/",34.0,OS.WINDOWS),
            SiteVisit("/",22.0,OS.MAC),
            SiteVisit("/login",12.0,OS.WINDOWS),
            SiteVisit("/signup",8.0,OS.IOS),
            SiteVisit("/",16.3,OS.ANDROID)
    )
    println(log.averageDurationFor(OS.WINDOWS)) //23.0
    println(log.averageDurationFor(OS.MAC)) //22.0
```

## 8.2 内联函数：消除lambda带来的运行时开销

### 8.2.1 内联函数如何运作

### 8.2.2 内联函数的限制

### 8.2.3 内联集合操作

### 8.2.4 决定何时将函数声明成内联

### 8.2.5 使用内联lambda管理资源

## 8.3 高阶函数中的控制流

### 8.3.1 lambda中的返回语句：从一个封闭的函数返回

### 8.3.2 从lambda返回：使用标签返回

### 8.3.3 匿名函数：默认使用局部返回

