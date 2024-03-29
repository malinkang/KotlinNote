---
title: 《Kotlin实战》第4章 类对象和接口
date: 2018-08-28 12:26:55
tags: ["Kotlin"]
---

## 4.1 定义类继承结构

### 4.1.1 Kotlin中的接口

```kotlin
//接口定义
interface Clickable {
    fun click()
}
//实现接口
class Button:Clickable{
    override fun click() = println("I was clicked")
}
```

接口的方法可以有一个默认实现。

```kotlin
interface Clickable {
    fun click()
    fun showOff() = println("I'm clickable!") //带默认实现的方法
}
```

假设存在同样定义了一个showOff方法并且有如下实现的另一个接口。

```kotlin
interface Focusable {
    fun setFocus(b: Boolean) =
            println("I ${if (b) "got" else "lost"} focus.")
    fun showOff() = println("I'm focusable!")
}
```

类实现这两个接口，如果没有显式实现showOff，将会得到如下编译错误：


```kotlin
class Button : Clickable,Focusable {
    override fun click() = println("I was clicked")
    override fun showOff() {
        super<Clickable>.showOff()
        super<Focusable>.showOff()
    }
}
```

### 4.1.2 open、final和abstarc修饰符：默认为final

Java的类和方法默认是open的，而kotlin中默认都是final的。如果你想允许创建一个类的子类， 需要使用open修饰符来标示这个类。此外，需要给每一个可以被重写的属性或方法添加open修饰符。

```kotlin
open class RichButton:Clickable{
    fun disable(){} //final函数
    open fun animate(){} //open函数
    override fun click() {} //这个函数重写了一个open函数它本身同样是open函数
}
```

如果重写了一个基类或者接口的成员，重写了的成员同样默认是open的，如果你想改变这一行为，阻止你的子类重写你的实现，可以显式地将重写的成员标注为final。

```kotlin
open class RichButton:Clickable{
    final override fun click() {} 
}
```

**抽象成员**始终是open的，所以不需要显式地使用open修饰符。

```kotlin
abstract class Animated{
    abstract fun animate()
    //抽象类中的非抽象函数并不是默认open的,但是可以标注为open的
    open fun stopAnimating(){}
    fun animateTwice(){}
}
```

### 4.1.3 可见性修饰符：默认为public

### 4.1.4 内部类和嵌套类：默认是嵌套类

与Java不同，Kotlin的嵌套类不能访问外部类的实例。

### 4.1.5 密封类：定义受限的类继承结构

```kotlin
interface Expr
class Num(val value: Int) : Expr
class Sum(val left: Expr, val right: Expr) : Expr

fun eval(e: Expr): Int =
        when (e) {
            is Num -> e.value
            is Sum -> eval(e.right) + eval(e.left)
            else -> //必须检查else分支
                throw IllegalArgumentException("Unknown expression")
        }
```

总是不得不添加一个默认分支很不方便。更重要的是，如果你添加一个新的子类，编译器并不能发现有地方改变了，如果你忘记了添加一个新分支，就会选择默认的选项，这可能导致潜在的bug。

Kotlin为这个问题提供了一个解决方案：sealed类。为父类添加一个sealed修饰符，对可能创建的子类做出严格的限制。所有的直接子类必须嵌套在父类中。

```kotlin
sealed class Expr {
    class Num(val value: Int) : Expr()
    class Sum(val left: Expr, val right: Expr) : Expr()
}

fun eval(e: Expr): Int =
        when (e) { //不需要提供默认分支
            is Expr.Num -> e.value
            is Expr.Sum -> eval(e.right) + eval(e.left)
        }
```

sealed修饰符隐含的这个类是一个open类，不再需要显式地添加open修饰符。

## 4.2 声明一个带非默认构造方法或属性的类

### 4.2.1 初始化类：主构造方法和初始化语句块

```kotlin
class User(val nickname: String)
```

这段被括号围起来的语句块就叫作主构造方法，它主要有两个目的：表明构造方法的参数，以及定义使用这些参数初始化的属性。

```kotlin
class User constructor(_nickname: String) {
    val nickname :String
    init {
        nickname = _nickname
    }
}
```

`constructor`关键字用来开始一个主构造方法或从构造方法的声明。`init`关键字用来引入一个初始化语句块。因为主构造方法有语法限制，不能包含初始化代码，所以要使用初始化语句块。一个类中可以声明多个初始化语句块。

构造方法参数`_nickname`中的下划线用来区分属性的名字和构造方法参数的名字。另一个可选方案是使用同样的名字，通过this来消除歧义。

```kotlin
class User constructor(nickname: String) {
    val nickname :String
    init {
        this.nickname = nickname
    }
}
```

在这个例子中，不需要把初始化代码块放在初始化语句块中，因为它可以与nickname属性的声明结合。如果主构造方法没有注解或可见性修饰符，同样可以去掉constructor关键字。

```kotlin
class User(_nickname: String) {
    val nickname = _nickname
}
```

```kotlin
class User(val nickname: String,val isSubscribed:Boolean = true) 
```

```kotlin
open class User(val nickname: String)

class TwitterUser(nickname: String) : User(nickname)

open class Button //将生成一个不带任何参数的默认构造方法

class RadioButton : Button() //注意与接口的区别,接口不带括号
```

如果不想类被其他代码实例化，必须把构造方法标记为`private`。

```kotlin
class Secretive private constructor() {}
```

###  4.2.2 构造方法：用不同的方式来初始化父类

```kotlin
open class View {
    constructor(ctx: Context){
        //...
    }
    constructor(ctx: Context,attr:AttributeSet){
        //...
    }
}
```

```kotlin
class MyButton : View {
    constructor(ctx: Context):super(ctx){ //调用父类构造方法
        //...
    }
    constructor(ctx: Context,attr:AttributeSet):super(ctx,attr){
        //...
    }
}
```

```kotlin
class MyButton : View {
    constructor(ctx: Context):this(ctx,MY_STY){ //委托给这个类的另一个构造方法
        //...
    }
    constructor(ctx: Context,attr:AttributeSet):super(ctx,attr){
        //...
    }
}
```

### 4.2.3 实现在接口中声明的属性

```kotlin
interface User {
    val nickname: String
}

class PrivateUser(override val nickname: String) : User

class SubscribingUser(val email: String) : User {
    override val nickname: String
        get() = email.substringBefore('@')
}

class FacebookUser(val accountId: Int) : User {
    override val nickname: String = getFacebookName(accountId) //属性初始化
    fun getFacebookName(accountId: Int): String {
        return ""
    }
}

```

### 4.2.4 通过getter或setter访问幕后字段


```kotlin
class User(val name:String){
    var address:String = "unspecified"
    set(value:String){  
        println("""
        Address was changed for $name:
        "$field" -> "$value".""".trimIndent()) //获取backing field的值
        field = value //更新支持字段的值
    }
}
```

### 4.2.5 修改访问器的可见性

```kotlin
class LengthCounter {
    //不能在类的外部修改这个属性
    var counter: Int = 0
        private set

    fun addWord(word: String) {
        counter += word.length
    }
}
```

```kotlin
val lengthCounter = LengthCounter()
lengthCounter.addWord("Hi!")
println(lengthCounter.counter)
```

## 4.3 编译器生成的方法：数据类和类委托

### 4.3.1 通用对象方法

```kotlin
class Client(val name:String,val postalCode:Int)
```


```kotlin
class Client(val name: String, val postalCode: Int) {
    override fun toString() = "Client(name=$name,postalCode=$postalCode)"
}
val client1 = Client("Alice",142562)
println(client1)
//Client(name=Alice,postalCode=142562)
```

#### 对象相等性：equals\(\)

```kotlin
val client1 = Client("Alice", 142562)
val client2 = Client("Alice", 142562)
//在Kotlin中,==检查对象是否相等而不是比较引用。这里会编译成调用"equals"
println(client1 == client2) //false
```

```kotlin
class Client(val name: String, val postalCode: Int) {
    override fun equals(other: Any?): Boolean {
        if (other == null || other !is Client)
            return false
        return name == other.name && postalCode == other.postalCode
    }

    override fun toString() = "Client(name=$name,postalCode=$postalCode)"
}
```

#### Hash容器：hashCode\(\)

```kotlin
class Client(val name: String, val postalCode: Int) {
    //...
    override fun hashCode(): Int = name.hashCode() * 31 + postalCode
}
```

### 4.3.2 数据类：自动生成通用方法的实现

```kotlin
data class Client(val name: String, val postalCode: Int)
```

```kotlin
val client1 = Client("Alice", 142562)
val client2 = Client("Alice", 142562)
println(client1) //Client(name=Alice, postalCode=142562)
println(client1 == client2) //true
val processed=hashSetOf(client1)
println(processed.contains(client2)) // true
```

#### 数据类和不可变性：copy\(\)方法

虽然数据类的属性并没有要求是val，但是强烈推荐只使用只读属性，让数据类的实例不可变。

为了让使用不可变对象的数据类变得更容易，Kotlin编译器为它们多生成了一个方法：一个允许copy类的实例的方法，并在copy的同时修改某些属性的值。创建副本通常是修改实例的好选择：副本有着单独的生命周期而且不会影响代码中引用原始实例的位置。

```kotlin
val bob = Client("Bob",973293)
println(bob.copy(postalCode = 382555)) //Client(name=Bob, postalCode=382555)
```

### 4.3.3 类委托：使用“by”关键字

```kotlin
class DelegatingCollection<T>: Collection<T> {
    private val innerList = arrayListOf<T>()
    override val size: Int = innerList.size
    override fun contains(element: T): Boolean = innerList.contains(element)

    override fun containsAll(elements: Collection<T>): Boolean = innerList.containsAll(elements)

    override fun isEmpty(): Boolean = innerList.isEmpty()

    override fun iterator(): Iterator<T> = innerList.iterator()
}
```

```kotlin
class DelegatingCollection<T>(innserList:Collection<T> = ArrayList<T>())
: Collection<T> by innserList
```

```kotlin
class CountingSet<T>(
        val innserSet: MutableCollection<T> = HashSet<T>()
) : MutableCollection<T> by innserSet {
    var objectsAdded = 0
    override fun add(element: T): Boolean {
        objectsAdded++
        return innserSet.add(element)
    }

    override fun addAll(elements: Collection<T>): Boolean {
        objectsAdded += elements.size
        return innserSet.addAll(elements)
    }
}
```

```kotlin
val cset = CountingSet<Int>()
cset.addAll(listOf(1,1,2))
println("${cset.objectsAdded}  objects were added, ${cset.size} remain") //3  objects were added, 2 remain
```

## 4.4 “object”关键字：将声明一个类与创建一个实例结合起来

Kotlin中object关键字在多种情况下出现，但是它们都遵循同样的核心概念：这个关键字定义一个类并同时创建一个实例。让我们来看看使用它的不同场景：

* 对象声明时定义单例的一种方式。
* 伴生对象可以持有工厂方法和其他与这个类相关，但在调用时并不依赖类实例的方法。它们的成员可以通过类名来访问。
* 对象表达式用来替代Java的匿名内部类。

### 4.4.1对象声明：创建单例易如反掌

```kotlin
object Payroll{
    val allEmployees = arrayListOf<Person>()
    fun calculateSalary(){
        for (person in allEmployees){

        }
    }
}
```

对象声明通过object关键字引入。一个对象声明可以非常高效地以一句话来定义一个类和一个该类的变量。

与类一样，一个对象声明也可以包含属性、方法、初始化语句块等的声明。唯一不允许的就是构造方法。与普通类的实例不同，对象声明在定义的时候就立即创建了，不需要在代码的其他地方调用构造方法。为对象声明定义一个构造方法是没有意义的。

```kotlin
Payroll.allEmployees.add(Person(...))
Payroll.calculateSalary()
```

对象声明同样可以继承自类和接口。

```kotlin
object CaseInsensitiveFileComparator:Comparator<File>{
    override fun compare(o1: File, o2: File): Int {
        return o1.path.compareTo(o2.path,ignoreCase = true)
    }
}
```

```kotlin
val files = listOf(File("/Z"),File("/a"))
println(files.sortedWith(CaseInsensitiveFileComparator)) //[/a, /Z]
```

同样可以在类中声明对象。

```kotlin
data class Person(val name: String) {
    object NameComparator : Comparator<Person> {
        override fun compare(o1: Person, o2: Person): Int =
                o1.name.compareTo(o2.name)
    }
}
```

```kotlin
val persons = listOf(Person("Bob"),Person("Alice"))
println(persons.sortedWith(Person.NameComparator))
```

### 4.4.2 伴生对象：工厂方法和静态成员的地盘

Kotlin中的类不能拥有静态成员。在类中定义的对象使用关键字：companion，就获得了直接通过容器类名称来访问这个对象的方法和属性的能力。

```kotlin
class A {
    companion object {
        fun bar() {
            println("Companion object called")
        }
    }
}
```

```kotlin
A.bar() //Companion object called
```

伴生对象可以访问类中的所有private成员，包括private构造方法，它是实现工厂模式的理想选择。

```kotlin
class User {
    val nickname:String
    constructor(email:String){
        nickname = email.substringBefore('@')
    }
    constructor(facebookAccountId:Int){
        nickname = getFacebookName(facebookAccountId)
    }
    fun getFacebookName(facebookAccountId:Int):String{
        var facebookName:String = ""
        //...
        return facebookName
    }
    
}
```

```kotlin
class User private constructor(val nickname:String){
    
    companion object{
        fun newSubscribingUser(email:String) = User(email.substringBefore('@'))
        fun newFacebookUser(accountId:Int) = User(getFacebookName(accountId))

        fun getFacebookName(facebookAccountId:Int):String{
            var facebookName:String = ""
            //...
            return facebookName
        }
    }
}
```

### 4.4.3 作为普通对象使用的伴生对象

伴生对象是一个声明在类中的普通对象。它可以有名字，实现一个接口或者有扩展函数或属性。

```kotlin
class Person(val name:String){
    companion object Loader{
        fun fromJson(jsonText:String):Person = ...
    }
}
```

```kotlin
interface JSONFactory<T> {
    fun fromJSON(jsonText: String):T
}
class Person(val name:String){
    companion object :JSONFactory<Person>{
        override fun fromJSON(jsonText: String): Person  = ...
    }
}
```

如果你有一个函数使用抽象方法来加载实体，可以传给它Person对象。

```kotlin
fun loadFromJSON<T>(factory:JSONFactory<T>):T{
    //...
}
loadFromJSON(Person) //Person类的名字被当做JSONFactory的实例
```

#### 伴生对象扩展

### 4.4.4 对象表达式：改变写法的匿名内部类

```kotlin
window.addMouseListener(
    object : MouseAdapter(){
        override fun mouseClicked(e:MouseEvent){
            //...
        }
        override fun mouseEntered(e:MouseEvent){
            //...
        }
    }
)
```

```kotlin
fun countClicks(window:Window){
    var clickCount = 0
    window.addMouseListener(object:MouseAdapter(){
        override fun mouseClicked(e:MouseEvent){
            clickCount++
        }
    }
}

```


## 更多阅读

* [类与继承](https://www.kotlincn.net/docs/reference/classes.html)