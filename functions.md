---
title: "Kotlin函数"
date: 2018-07-27T15:05:51+08:00
lastmod: 2018-07-27T15:05:51+08:00
draft: false
tags: ["Kotlin"]
toc: true
---

## 函数声明

`Kotlin`中的函数使用`fun`关键字声明：

```kotlin
fun double(x: Int): Int {
    return 2 * x
}
```

## 函数调用

调用函数使用传统的方法：
```kotlin
val result = double(2)
```
调用成员函数使用点表示法：
```kotlin
Stream().read() // 创建类 Stream 实例并调用 read()
```

## 参数

函数参数使用`Pascal`表示法定义，即 `name: type`。参数用逗号隔开。每个参数必须有显式类型：

```kotlin
fun powerOf(number: Int, exponent: Int) { /*……*/ }
```
### 默认参数

函数参数可以有默认值，当省略相应的参数时使用默认值。与其他语言相比，这可以减少重载数量：

```kotlin
fun read(b: Array<Byte>, off: Int = 0, len: Int = b.size) { /*……*/ }
```
默认值通过类型后面的`=`及给出的值来定义。


覆盖方法总是使用与基类型方法相同的默认参数值。 当覆盖一个带有默认参数值的方法时，必须从签名中省略默认参数值：

```kotlin
open class A {
    open fun foo(i: Int = 10) { /*……*/ }
}

class B : A() {
    //如果有默认值，编译器将会报错An overriding function is not allowed to specify default values for its parameters
    override fun foo(i: Int) { /*……*/ }  // 不能有默认值
}
```
如果一个默认参数在一个无默认值的参数之前，那么该无默认值只能通过使用具名参数调用该函数来使用：

```kotlin
fun foo(bar: Int = 0, baz: Int) { /*……*/ }

foo(baz = 1) // 使用默认值 bar = 0
```

如果在默认参数之后的最后一个参数是`lambda`表达式，那么它既可以作为具名参数在括号内传入，也可以在括号外传入：

```kotlin
fun foo(bar: Int = 0, baz: Int = 1, qux: () -> Unit) { /*……*/ }

foo(1) { println("hello") }     // 使用默认值 baz = 1
foo(qux = { println("hello") }) // 使用两个默认值 bar = 0 与 baz = 1
foo { println("hello") }        // 使用两个默认值 bar = 0 与 baz = 1
```

当你从`Java`中调用`Kotlin`函数的时候必须显式地指定所有参数值。可以用@JvmOverloads注解它。编译器会生成重载函数。

### 具名参数

可以在调用函数时使用具名的函数参数。当一个函数有大量的参数或默认参数时这会非常方便。

给定以下函数：

```kotlin
fun reformat(str: String,
             normalizeCase: Boolean = true,
             upperCaseFirstLetter: Boolean = true,
             divideByCamelHumps: Boolean = false,
             wordSeparator: Char = ' ') {
/*……*/
}
```
我们可以使用默认参数来调用它：

```kotlin
reformat(str)
```
然而，当使用非默认参数调用它时，该调用看起来就像：

```kotlin
reformat(str,
    normalizeCase = true,
    upperCaseFirstLetter = true,
    divideByCamelHumps = false,
    wordSeparator = '_'
)
```
并且如果我们不需要所有的参数：

```kotlin
reformat(str, wordSeparator = '_')
```

当一个函数调用混用位置参数与具名参数时，所有位置参数都要放在第一个具名参数之前。

```kotlin
fun f(x: Int = 0, y: Int = 0){/*……*/}
// f(x=1,2) Mixing named and positioned arguments is not allowed
f(1,2)
f(1,y=2)
```
可以通过使用星号操作符将可变数量参数（vararg） 以具名形式传入：
```kotlin
fun foo(vararg strings: String) { /*……*/ }

foo(strings = *arrayOf("a", "b", "c"))
```
### 可变数量的参数

函数的参数（通常是最后一个）可以用 vararg 修饰符标记：

```kotlin
fun <T> asList(vararg ts: T): List<T> {
    val result = ArrayList<T>()
    for (t in ts) // ts is an Array
        result.add(t)
    return result
}
```
允许将可变数量的参数传递给函数：
```kotlin
val list = asList(1, 2, 3)
```

在函数内部，类型 T 的 vararg 参数的可见方式是作为 T 数组，即上例中的 ts 变量具有类型 Array <out T>。

只有一个参数可以标注为 vararg。如果 vararg 参数不是列表中的最后一个参数， 可以使用具名参数语法传递其后的参数的值，或者，如果参数具有函数类型，则通过在括号外部传一个 lambda。

当我们调用 vararg-函数时，我们可以一个接一个地传参，例如 asList(1, 2, 3)，或者，如果我们已经有一个数组并希望将其内容传给该函数，我们使用伸展（spread）操作符（在数组前面加 *）：

```kotlin
val a = arrayOf(1, 2, 3)
val list = asList(-1, 0, *a, 4)
```

## 返回值

### 返回 Unit 的函数

如果一个函数不返回任何有用的值，它的返回类型是 Unit。Unit 是一种只有一个值——Unit 的类型。这个值不需要显式返回：

```kotlin
fun printHello(name: String?): Unit {
    if (name != null)
        println("Hello $name")
    else
        println("Hi there!")
    // `return Unit` 或者 `return` 是可选的
}
```

`Unit`返回类型声明也是可选的。上面的代码等同于：

```kotlin
fun printHello(name: String?) { …… }
```

### 单表达式函数

当函数返回单个表达式时，可以省略花括号并且在 = 符号之后指定代码体即可：

```kotlin
fun double(x: Int): Int = x * 2
```
当返回值类型可由编译器推断时，显式声明返回类型是可选的：

```kotlin
fun double(x: Int) = x * 2
```
### 显式返回类型

具有块代码体的函数必须始终显式指定返回类型，除非他们旨在返回`Unit`，在这种情况下它是可选的。 `Kotlin`不推断具有块代码体的函数的返回类型，因为这样的函数在代码体中可能有复杂的控制流，并且返回类型对于读者（有时甚至对于编译器）是不明显的。

## 扩展函数

`Kotlin`能够扩展一个类的新功能而无需继承该类或者使用像装饰者这样的设计模式。 这通过叫做`扩展`特殊声明完成。 例如，你可以为一个你不能修改的、来自第三方库中的类编写一个新的函数。 这个新增的函数就像那个原始类本来就有的函数一样，可以用普通的方法调用。 这种机制称为 扩展函数 。

声明一个扩展函数，我们需要用一个`接收者类型`也就是被扩展的类型来作为他的前缀。

```kotlin
fun String.lastChar(): Char = this.get(this.length-1)
println("Kotlin".lastChar()) // n
```
在扩展函数中，可以像其他成员函数一样用`this`。而且也可以像普通的成员函数一样，省略它。

```kotlin
fun String.lastChar(): Char = get(length-1)
```
在扩展函数中，可以直接访问被扩展的类的其他方法和属性，就好像是在这个类自己的方法中访问它们一样。和在类内部定义的方法不同的是，扩展函数不能访问私有的或者受保护的成员。

### 扩展是静态解析的

扩展不能真正的修改他们所扩展的类。通过定义一个扩展，你并没有在一个类中插入新成员， 仅仅是可以通过该类型的变量用点表达式去调用这个新函数。

我们想强调的是扩展函数是静态分发的，即他们不是根据接收者类型的虚方法。 这意味着调用的扩展函数是由函数调用所在的表达式的类型来决定的， 而不是由表达式运行时求值结果决定的。例如：
```kotlin
open class Shape

class Rectangle: Shape()

fun Shape.getName() = "Shape"

fun Rectangle.getName() = "Rectangle"

fun printClassName(s: Shape) {
    println(s.getName())
}    

printClassName(Rectangle())
```

这个例子会输出 "Shape"，因为调用的扩展函数只取决于参数 s 的声明类型，该类型是 Shape 类。

如果一个类定义有一个成员函数与一个扩展函数，而这两个函数又有相同的接收者类型、 相同的名字，并且都适用给定的参数，这种情况总是取成员函数。 例如：

```kotlin
class Example {
    fun printFunctionType() { println("Class method") }
}

fun Example.printFunctionType() { println("Extension function") }

Example().printFunctionType()
```


这段代码输出“Class method”。

当然，扩展函数重载同样名字但不同签名成员函数也完全可以：

```kotlin
class Example {
    fun printFunctionType() { println("Class method") }
}

fun Example.printFunctionType(i: Int) { println("Extension function") }

Example().printFunctionType(1)
```
## 中缀表示法

标有`infix`关键字的函数也可以使用中缀表示法（忽略该调用的点与圆括号）调用。中缀函数必须满足以下要求：

* 它们必须是成员函数或扩展函数；
* 它们必须只有一个参数；
* 其参数不得接受可变数量的参数且不能有默认值。

```kotlin
infix fun Int.shl(x: Int): Int { …… }

// 用中缀表示法调用该函数
1 shl 2

// 等同于这样
1.shl(2)
```
中缀函数调用的优先级低于算术操作符、类型转换以及 rangeTo 操作符。 以下表达式是等价的：
```kotlin
1 shl 2 + 3 等价于 1 shl (2 + 3)
0 until n * 2 等价于 0 until (n * 2)
xs union ys as Set<*> 等价于 xs union (ys as Set<*>)
```
另一方面，中缀函数调用的优先级高于布尔操作符 && 与 ||、is- 与 in- 检测以及其他一些操作符。这些表达式也是等价的：
```kotlin
a && b xor c 等价于 a && (b xor c)
a xor b in c 等价于 (a xor b) in c
```
请注意，中缀函数总是要求指定接收者与参数。当使用中缀表示法在当前接收者上调用方法时，需要显式使用 this；不能像常规方法调用那样省略。这是确保非模糊解析所必需的。

```kotlin
class MyStringCollection {
    infix fun add(s: String) { /*……*/ }
    
    fun build() {
        this add "abc"   // 正确
        add("abc")       // 正确
        //add "abc"        // 错误：必须指定接收者
    }
}
```
## 函数作用域

在`Kotlin`中函数可以在文件顶层声明，这意味着你不需要像一些语言如 Java、C# 或 Scala 那样需要创建一个类来保存一个函数。此外除了顶层函数，`Kotlin`中函数也可以声明在局部作用域、作为成员函数以及扩展函数。

### 局部函数
Kotlin 支持局部函数，即一个函数在另一个函数内部：

```kotlin
fun dfs(graph: Graph) {
    fun dfs(current: Vertex, visited: MutableSet<Vertex>) {
        if (!visited.add(current)) return
        for (v in current.neighbors)
            dfs(v, visited)
    }

    dfs(graph.vertices[0], HashSet())
}
```
局部函数可以访问外部函数（即闭包）的局部变量，所以在上例中，visited 可以是局部变量：

```kotlin
fun dfs(graph: Graph) {
    val visited = HashSet<Vertex>()
    fun dfs(current: Vertex) {
        if (!visited.add(current)) return
        for (v in current.neighbors)
            dfs(v)
    }

    dfs(graph.vertices[0])
}
```

```kotlin
//报错 Anonymous functions with names are prohibited 
val transform:(Int)->Int = fun foo(i:Int):Int{
    return i*i;
};
```

## 参考

* [函数](https://www.kotlincn.net/docs/reference/functions.html)
* [扩展](https://www.kotlincn.net/docs/reference/extensions.html)