---
title: "Kotlin作用域函数"
date: 2018-07-27T15:05:51+08:00
tags: ["Kotlin"]
---

`Kotlin`标准库包含几个函数，它们的唯一目的是在对象的上下文中执行代码块。当对一个对象调用这样的函数并提供一个`lambda`表达式时，它会形成一个临时作用域。在此作用域中，可以访问该对象而无需其名称。这些函数称为作用域函数。共有以下五种：`let`、`run`、`with`、`apply` 以及`also`。

这些函数基本上做了同样的事情：在一个对象上执行一个代码块。不同的是这个对象在块中如何使用，以及整个表达式的结果是什么。


由于作用域函数本质上都非常相似，因此了解它们之间的区别很重要。每个作用域函数之间有两个主要区别：

* 引用上下文对象的方式
* 返回值

| 函数    | 对象引用 | 返回值            | 是否是扩展函数             |
| ------- | -------- | ----------------- | -------------------------- |
| `let`   | `it`     | Lambda 表达式结果 | 是                         |
| `run`   | `this`   | Lambda 表达式结果 | 是                         |
| `run`   | -        | Lambda 表达式结果 | 不是：调用无需上下文对象   |
| `with`  | `this`   | Lambda 表达式结果 | 不是：把上下文对象当做参数 |
| `apply` | `this`   | 上下文对象        | 是                         |
| `also`  | `it`     | 上下文对象        | 是                         |


```java
data class Person(val name: String, var age: Int, var city: String) {
    fun moveTo(city: String) {
        this.city = city
    }

    fun incrementAge() {
        age++
    }
}
```

```kotlin
//let
val p1 = Person("Alice", 20, "Amsterdam").let {
    it.moveTo("London")
    it.incrementAge()
    it
}
// apply
println(p1)
val p2 = Person("Alice", 20, "Amsterdam").apply {
    moveTo("London")
    incrementAge()
}
println(p2)
// also
val p3 = Person("Alice", 20, "Amsterdam").also {
    it.moveTo("London")
    it.incrementAge()
}
println(p3)
// with
val p4 = with(Person("Alice", 20, "Amsterdam")) {
    moveTo("London")
    incrementAge()
    this
}
println(p4)
//run 
//run和with两个很相似，区别run是扩展函数 with不是
val p5 = Person("Alice", 20, "Amsterdam").run {
    moveTo("London")
    incrementAge()
    this
}
println(p5)
//run
val p6 = run {
    val person = Person("Alice", 20, "Amsterdam")
    person.moveTo("London")
    person.incrementAge()
    person
}
println(p6)
```

## 参考

* [作用域函数](https://www.kotlincn.net/docs/reference/scope-functions.html)