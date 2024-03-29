---
title: 《Kotlin实战》读书笔记 第3章 函数的定义与调用
date: 2018-08-25 12:26:55
tags: ["Kotlin"]
---

## 3.1 在kotlin中创建集合

```kotlin
val set = hashSetOf(1, 7, 53)
val list = arrayListOf(1, 7, 53)
val map = hashMapOf(1 to "one", 7 to "seven", 53 to "fifty-three")
//kotlin的javaClass等价于Java的getClass()
println(set.javaClass)  //class java.util.HashSet
println(list.javaClass) //class java.util.ArrayList
println(map.javaClass) //class java.util.HashMap
```

`Kotlin`没有采用它自己的集合类，而是采用的标准的`Java`集合类。`Kotlin`可以更容易与Java代码交互。当从`Kotlin`中调用`Java`函数的时候，不用转换它的集合类来匹配`Java`的类，反之亦然。

```kotlin
val strings = listOf("first","second","fourteenth")
//获取最后一个元素
println(strings.last())
val numbers = setOf(1,14,2)
//得到一个数字列表的最大值
println(numbers.max())
```

## 3.2 让函数更好调用

```kotlin
fun <T> joinToString(collection:Collection<T>,separator:String,
prefix:String,postfix:String):String{
    val result = StringBuilder(prefix)
    for ((index,element) in collection.withIndex()){
        if(index>0) result.append(separator)
        result.append(element)
    }
    result.append(postfix)
    return result.toString()
}
```

```kotlin
val list = listOf(1,2,3)
println(joinToString(list,";","(",")"))
```

### 3.2.1 命名参数

当调用一个`kotlin`定义的函数时，可以显式地标明一些参数的名称。如果在调用一个函数时，指明一个参数的名称，为了避免混淆，那它之后的所有参数都要标明名称。当调用Java的函数时，不能采用命名参数。

```kotlin
 println(joinToString(list,separator =";",prefix = "(",postfix = ")"))
```

### 3.2.2 默认参数值

在`Kotlin`中，可以在声明函数的时候，指定参数的默认值，这样就可以避免创建重载的函数。

```kotlin
fun <T> joinToString(collection:Collection<T>,
                     separator:String = ",",
                     prefix:String = "",
                     postfix:String = ""):String{
   //...
}

```

```kotlin
println(joinToString(list)) // 1,2,3
println(joinToString(list,";")) //1;2;3
println(joinToString(list,";","(",")")) //(1;2;3)
println(joinToString(list,prefix = "(")) //(1,2,3
```

当使用常规的调用语法时，必须按照函数声明中定义的参数顺序来给定参数，可以省略的只有排在末尾的参数。如果使用命名参数，可以省略中间的一些参数，也可以以你想要的任意顺序只给定你需要的参数：

当你从`Java`中调用`Kotlin`函数的时候必须显式地指定所有参数值。如果需要从Java代码中做频繁的调用，而且希望它能对`Java`的调用者更简便，可以用`@JvmOverloads`注解它。这个指示编译器生成如下重载函数。

扩展函数需要进行导入才能使用它。


### 3.2.3 消除静态工具类：顶层函数和属性

```kotlin
package strings
fun joinToString(...): String{...}
```
这里它会编译成如下的Java代码：

```java
package strings;
public class JoinKt {
    public static String joinToString(...){...}
}
```
可以看到Kotlin编译生成的类的名称，对应于包含函数的文件的名称。这个文件中的所有顶层函数编译为这个类的静态函数。

要改变包含Kotlin顶层函数的生成的类的名称，需要为这个文件添加@JvmName的注解，将其放到这个文件的开头，位于包名的前面：

```kotlin
@file:JvmName("StringFunctions")
package strings
```

#### 顶层属性

```kotlin
const val UNIX_LINE_SEPARATOR = "\n"
```

## 3.3 扩展函数和属性

添加扩展函数就是把你要扩展的类或者接口的名称，放到即将添加的函数前面。这个类的名称被称为接收者类型；用来调用这个扩展函数的那个对象，叫做接收者对象。

```kotlin
fun String.lastChar(): Char = this.get(this.length-1)
```

```kotlin
println("Kotlin".lastChar()) // n
```

在扩展函数中，可以像其他成员函数一样用this。而且也可以像普通的成员函数一样，省略它。

```kotlin
fun String.lastChar(): Char = get(length-1)
```

在扩展函数中，可以直接访问被扩展的类的其他方法和属性，就好像是在这个类自己的方法中访问它们一样。和在类内部定义的方法不同的是，扩展函数不能访问私有的或者受保护的成员。 

### 3.3.1 导入和扩展函数

```kotlin
import strings.lastChar
//import strings.* //也可以用*来导入
println("Kotlin".lastChar()) // n
```

可以使用关键字as来修改导入的类或者函数名称，可以避免重名函数的冲突。

```kotlin
import strings.lastChar as last
println("Kotlin".last()) // n
```

### 3.3.2 从Java中调用扩展函数



和顶层函数一样，包含这个函数的Java类的名称，是由这个函数声明的文件名称决定的。假设它声明在一个叫做StringUtil.kt的文件中

```java
char c = StringUtilKt.lastChar("Java");
```

### 3.3.3 作为扩展函数的工具函数

```kotlin
fun <T> Collection<T>.joinToString(
                     separator:String = ",",
                     prefix:String = "",
                     postfix:String = ""):String{

    val result = StringBuilder(prefix)
    for ((index,element) in withIndex()){
        if(index>0) result.append(separator)
        result.append(element)
    }
    result.append(postfix)
    return result.toString()
}
```

```kotlin
val list = listOf(1,2,3)
println(list.joinToString(separator = ";",prefix = "(",postfix = ")")) //(1;2;3)
```

扩展函数的静态性质也决定了扩展函数不能被子类重写。

### 3.3.4 不可重写的扩展函数

```kotlin
open class View {
    open fun click() = println("View clicked")
}
```

```kotlin
open class Button:View() {
    override fun click() = println("Button clicked")
}
```

```kotlin
val view:View = Button()
view.click() //Button clicked
```

```kotlin
fun View.showoff() = println("I'm a view!")
fun Button.showoff() = println("I'm a button!")
val view:View = Button()
view.showoff() //I'm a view!
```

### 3.3.4 扩展属性

```kotlin
val String.lastChar: Char
    get() = get(length - 1)
```
```kotlin
println("Kotlin".lastChar) //n
val sb = StringBuilder("Kotlin?") 
sb.lastChar = '!'
println(sb) //Kotlin!
```

## 3.4 处理集合：可变参数、中缀调用和库的支持

### 3.4.1 扩展Java集合的API

### 3.4.2 可变参数：让函数支持任意数量的参数

Kotlin可变参数是在参数上使用vararg修饰符。

```kotlin
public fun <T> listOf(vararg elements: T): List<T>{...}
```

在`Java`中可以按原样传递数组，而`kotlin`则要求你显式地解包数组，以便每个数组元素在函数中能作为单独的参数来调用。从技术的角度来讲，这个功能被称为`展开运算符`。

```kotlin
fun main(args: Array<String>) {
    val list = listOf("args:",*args)
    println(list) //[args:, one, two, three]
}
```

### 3.4.3 键值对的处理：中缀调用和解构声明

```kotlin
 val map = hashMapOf(1 to "one", 7 to "seven", 53 to "fifty-three")
```

这行代码中的单词to不是内置的结构，而是一种特殊的函数调用，被称为中缀调用。

在中缀调用中，没有添加额外的分隔符，函数名称是直接放在目标对象名称和参数之间的。以下两种调用方式是等价的：

```kotlin
1.to("one")
1 to "one"
```

中缀调用可以与只有一个参数的函数一起使用，无论是普通的函数还是扩展函数。要允许使用中缀符号调用函数，需要使用infix修饰符来标记它。

```kotlin
public infix fun <A, B> A.to(that: B): Pair<A, B> = Pair(this, that)
```

可以直接用Pair的内容来初始化两个变量：

```kotlin
val (number, name) = 1 to "one"
```

这个功能称为解构声明。用to函数创建一个pair，然后用解构声明来展开。

## 3.5 字符串和正则表达式的处理

### 3.5.1 分割字符串

```java
String[] strings="12.345-6.A".split(".");
System.out.println(strings.length); //0
```

Java的split方法将一个正则表达式作为参数，并根据表达式将字符串分割成多个字符串。这里点（.）是表示任何字符的正则表达式。

Kotlin把这个令人费解的函数隐藏了，作为替换，提供了一些名为split的，具有不同参数的重载的扩展函数。用来承载正则表达式的值需要一个Regex类型，而不是String，这样确保了当有一个字符串传递给这些函数的时候，不会被当做正则表达式。

```kotlin
println("12.345-6.A".split("\\.|-".toRegex())) //[12, 345, 6, A]
```

Kotlin中的split扩展函数的其他重载支持任意数量的纯文本字符串分隔符：

```kotlin
println("12.345-6.A".split(".","-")) //[12, 345, 6, A]
```

### 3.5.2 正则表达式和三重引号的字符串

```kotlin
fun parsePath(path: String) {
    val directory = path.substringBeforeLast("/")
    val fullName = path.substringAfterLast("/")
    val fileName = fullName.substringBeforeLast(".")
    val extension = fullName.substringAfterLast(".")
    println("Dir: $directory, name: $fileName, ext: $extension")
}
```

### 3.5.3 多行三重引号的字符串

## 3.6 让你的代码更简洁：局部函数和扩展

```kotlin
class User(val id: Int, val name: String, val address: String)

fun saveUser(user:User){
    //重复的字段检查
    if(user.name.isEmpty()){
        throw IllegalArgumentException(
                "Can't save user ${user.id}:empty Name")
    }
    if(user.address.isEmpty()){
        throw IllegalArgumentException(
                "Can't save user ${user.id}:empty Address")
    }
    //保存user到数据库
}
```

```kotlin
fun saveUser(user: User) {
    fun validate(user: User,value: String, fieldName: String){
        if (value.isEmpty()) {
            throw IllegalArgumentException(
                    "Can't save user ${user.id}:empty $fieldName")
        }
    }
    validate(user,user.name,"Name")
    validate(user,user.address,"Address")
    //保存user到数据库
}
```

因为局部函数可以访问所在函数中的所有参数和变量。我们可以利用这一点，去冗余的User参数。

```kotlin
fun saveUser(user: User) {
    fun validate(value: String, fieldName: String){
        if (value.isEmpty()) {
            throw IllegalArgumentException(
                    "Can't save user ${user.id}:empty $fieldName")
        }
    }
    validate(user.name,"Name")
    validate(user.address,"Address")
    //保存user到数据库
}
```

```kotlin
fun User.validateBeforeSave(){
    fun validate(value: String, fieldName: String){
        if (value.isEmpty()) {
            throw IllegalArgumentException(
                    "Can't save user $id:empty $fieldName")
        }
    }
    validate(name,"Name")
    validate(address,"Address")
}
fun saveUser(user: User) {
    user.validateBeforeSave()
    //保存user到数据库
}
```

