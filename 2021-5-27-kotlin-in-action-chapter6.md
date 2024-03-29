---
title: 《Kotlin实战》第6章 Kotlin的类型系统
date: 2018-09-06 12:26:55
tags: ["Kotlin"]
---

## 6.1 可空性

### 6.1.1 可空类型


```kotlin
//增加了null检查后，这段代码就可以编译了
fun strLenSafe(s: String?) = if (s != null) s.length else 0
```

### 6.1.2 类型的含义

### 6.1.3 安全调用运算符

`安全调用运算符?`允许把一次null检查和一次方法调用合并成一个操作。

```kotlin
fun strLenSafe(s: String?) = s?.length
println(strLenSafe(null)) //null
```

### 6.1.4 Elvis运算符

Elvis运算符接收两个运算数，如果第一个运算数不为null，运算结果就是第一个运算数；如果第一个运算数为null，运算结果就是第二个运算数。

```kotlin
fun strLenSafe(s: String?) = s?.length
println(strLenSafe(null)) //null·
```

### 6.1.5 安全转换：“as?”

as?运算符尝试把值转换成指定的类型，如果值不是合适的类型就返回null。

### 6.1.6 非空断言：“!!”

### 6.1.7 “let” 函数

```kotlin
fun sendEmailTo(email:String){/**/}
```

```kotlin
if(email!=null) sendEmailTo(email)
```

let函数只有email的值非空时才被调用，所以你就能在lambda中把email当做非空的实参使用。

```kotlin
val email:String? = null
email?.let { email->sendEmailTo(email) }
email?.let(::sendEmailTo)
```

### 6.1.8 延迟初始化的属性

### 6.1.9 可空类型的扩展

### 6.1.10 类型参数的可空性

```kotlin
fun <T> printHashCode(t:T){
    println(t?.hashCode()) //t可能为null,所以必须使用安全调用
}
```


```kotlin
fun <T:Any> printHashCode(t:T){
    println(t.hashCode()) // t不是可空的
}
```
### 6.1.11 可控性和Java

## 6.2 基本数据类型和其他基本类型

### 6.2.1 基本数据类型：Int、Boolean及其他

Kotlin并不区分基本数据类型和包装类型，你使用的永远是同一个类型。

大多数情况下对于变量、属性、参数和返回类型，Kotlin的Int类型会被编译成Java基本数据类型int。唯一不可行的例外是泛型类，比如集合。用作泛型类型参数的基本数据类型会被编译成对应的Java包装类。

### 6.2.2 可控的基本数据类型：Int?、Boolean？及其他

Kotlin中的可空类型会编译成对应的包装类型。

### 6.2.3 数字转换

Kotlin不会自动地把数字从一个类型转换成另外一种，即便是转换成范围更大的类型。


```kotlin
val i = 0
val l: Long = i.toLong() //显式进行转换
```

### 6.2.4 “Any”和“Any?”：根类型

### 6.2.5 Unit类型：Kotlin的“void”

### 6.2.6 Nothing类型：“这个函数永不返回”

## 6.3 集合与数组

### 6.3.1 可空性和集合

### 6.3.2 只读集合与可变集合

`kotlin.collections.Collection`接口可以遍历集合中的元素、获取集合大小、判断集合中是否包含某个元素，以及执行其他从集合中读取数据的操作。但这个接口没有任何添加或移除元素的方法。

`kotlin.collections.MutableCollection`接口可以修改集合中的数据。它继承了普通的`kotlin.collections.Collection`接口。 还提供了方法来添加和移除元素、清空集合等。

### 6.3.3 Kotlin集合和Java

### 6.3.4 作为平台类型的集合

### 6.3.5 对象和基本数据类型的数组

Kotlin有以下方法来创建数组：

* arrayOf函数创建一个数组，它包含的元素是指定为该函数的实参
* arrayOfNulls创建一个给定大小的数组，包含的是null元素。当然，它只能用来创建包含元素类型可空的数组。
* Array构造方法接收数组的大小和一个lambda表达式，调用lambda表达式来创建每一个数组元素。这就是使用非空元素类型来初始化数组，但不用显式地传递每个元素的方式。

```kotlin
val letters = Array<String>(26){i->('a'+i).toString()}
println(letters.joinToString("")) //abcdefghijklmnopqrstuvwxyz
```

```kotlin
val strings = listOf("a","b","c")
//toTypedArray方法将集合转换为数组
println("%s/%s/%s".format(*strings.toTypedArray()))
```

要创建一个基本类型的数组，有如下选择：

* 该类型的构造方法接收size参数并返回一个使用对应基本数据类型默认值初始化好的数组。
* 工厂函数接收变长参数的值并创建存储这些值的数组
* 另一种构造方法，接收一个大小和一个用来初始化每个元素的lambda。

```kotlin
val fiveZeros = IntArray(5)
val fiveZerosToo = intArrayOf(0,0,0,0,0)
val squares = IntArray(5) { i -> (i + 1) * (i + 1) }
println(squares.joinToString()) //1, 4, 9, 16, 25
```

Kotlin标准库支持一套和集合相同的用于数组的扩展函数。

```kotlin
println(squares.filter { it % 2 == 0 }.joinToString()) //4, 16
```



* [Collections and sequences in Kotlin](https://medium.com/androiddevelopers/collections-and-sequences-in-kotlin-55db18283aca)