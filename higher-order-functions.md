# 高阶函数

高阶函数就是以另一个函数作为参数或者返回值的函数。

## 函数类型

![Kotlin&#x4E2D;&#x51FD;&#x6570;&#x7C7B;&#x578B;&#x8BED;&#x6CD5;](.gitbook/assets/image%20%286%29.png)

```kotlin
//函数类型的变量
val sum = { x: Int, y: Int -> x + y }
val action = { println(42)}
println(sum(1,2)) //3
action() //42
```

```kotlin
val sum: (Int, Int) -> Int = { x, y -> x + y } // 有两个Int型参数和Int型返回值的函数
val action: () -> Unit = { println(42) } //没有参数和返回值的函数 注意Unit不能省略
```

=== "Kotlin"
    ```kotlin
    class Person{
    val sayHi={print("hi ${this} ")}
    }
    fun main() {
        val p = Person()
        println(p)
        val hi = p.sayHi;
        hi();
    }
    //Person@4fca772d
    //hi Person@4fca772d 
    ```
=== "JavaScript"
    ```javascript
    class Person{
        sayHi(){
            console.log(`Hi ${this}`);
        }
    }
    const p = new Person();
    p.sayHi();
    const sayHi = p.sayHi;
    sayHi();
    Hi [object Object]
    Hi undefined
    ```

## 带有接收者的函数类型

Kotlin 提供了调用带有接收者（提供接收者对象）的函数类型实例的能力。

在这样的函数字面值内部，传给调用的接收者对象成为隐式的this，以便访问接收者对象的成员而无需任何额外的限定符，亦可使用 this 表达式 访问接收者对象。

这种行为与扩展函数类似，扩展函数也允许在函数体内部访问接收者对象的成员。

这里有一个带有接收者的函数字面值及其类型的示例，其中在接收者对象上调用了 plus ：

```kotlin
val sum: Int.(Int) -> Int = { other -> plus(other) }
3.sum(1)
```

匿名函数语法允许你直接指定函数字面值的接收者类型。 如果你需要使用带接收者的函数类型声明一个变量，并在之后使用它，这将非常有用。

```kotlin
val sum = fun Int.(other: Int): Int = this + other
```

## 调用作为参数的函数

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

## 在Java中使用函数类

```kotlin
LambdaTestKt.twoAndThree((a, b) -> a + b); //The result is 5
LambdaTestKt.twoAndThree((a, b) -> a * b); //The result is 6
```

## 函数类型的参数默认值和null值

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

## 返回函数的函数

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

## 通过lambda去除重复代码

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

