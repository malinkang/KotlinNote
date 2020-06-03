# 第5章 Lambda编程

## 5.1 Lambda表达式和成员引用

### 5.1.1 Lambda简介：作为函数参数的代码块

```java
button.setOnClickLisener(new OnClickListener(){
    @Override
    public void onClick(View view){
        //点击后执行的动作
    }
}
```

```kotlin
button.setOnClickListener(/*点击后执行的动作*/)
```

### 5.1.2 Lambda和集合

```kotlin
data class Person(val name:String,val age:Int)

fun findTheOldest(people:List<Person>){
    var maxAge = 0
    var theOldest:Person? = null
    for (person in people){
        if(person.age > maxAge){
            maxAge = person.age
            theOldest = person
        }
    }
    println(theOldest)
}
```

```kotlin
var people = listOf(Person("Alice",29),Person("Bob",31))
findTheOldest(people)
```

{% code-tabs %}
{% code-tabs-item title="用lambda在集合中搜索" %}
```kotlin
var people = listOf(Person("Alice",29),Person("Bob",31))
println(people.maxBy { it.age })
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 5.1.3 Lambda表达式的语法

![](../.gitbook/assets/image%20%282%29.png)

```kotlin
val sum = { x: Int, y: Int -> x + y }
println(sum(1,2))
{ println(42) }() //直接调用lambda表达式
run { println(42) } //使用库函数run来执行lambda
```

```kotlin
//不用任何简明语法来重写这个例子
people.maxBy({p:Person->p.age})
//如果lambda表达式是函数调用的最后一个实参，可以放到括号的外边
people.maxBy(){p:Person->p.age}
//当lambda是唯一的实参时，还可以去掉调用代码中的空括号对
people.maxBy { p:Person->p.age }
```

和局部变量一样，如果lambda参数的类型可以被推倒出来，你就不需要显式地指定它。以这里的maxBy函数为例，其参数类型始终和集合的元素类型相同。编译器知道你是对一个Person对象的集合调用maxBy函数，所以它能推断lambda参数也会是Person类型。

```kotlin
people.maxBy { p->p.age } //推导出参数类型
people.maxBy{ it.age }
```

如果当前上下文期望的是只有一个参数的Lambda切这个参数的类型可以推断出来，就会生成it这个名称。

{% code-tabs %}
{% code-tabs-item title="包含更多语句的lambda表达式" %}
```kotlin
val sum = {x:Int,y:Int->
    println("Computing the sum of $x and $y...")
    x + y
}
println(sum(1,2)) 
//Computing the sum of 1 and 2...
//3
```
{% endcode-tabs-item %}
{% endcode-tabs %}

### 5.1.4 在作用域中访问变量

```kotlin
fun printMessagePrefix(messages: Collection<String>, prefix: String) {
    messages.forEach {
        println("$prefix $it")
    }
}
val errors = listOf("403 Forbidden", "404 Not Found")
printMessagePrefix(errors, "Error:")
//Error: 403 Forbidden
//Error: 404 Not Found
```

```kotlin
fun printProblemCounts(response: Collection<String>) {
    var clientErrors = 0
    var serverErrors = 0
    response.forEach {
        if (it.startsWith("4")) {
            clientErrors++ //在lambda中修改变量
        } else if (it.startsWith("5")) {
            serverErrors++
        }
    }
    println("$clientErrors client errors,$serverErrors server errors")
}
val responses = listOf("200 OK", "418 I'm a teapot", "500 Internal Server Error")
printProblemCounts(responses)
```

和Java不一样，Kotlin允许在lambda内部访问非final变量甚至修改它们。从lambda内访问外部变量，我们称这些变量被lambda捕捉。

### 5.1.5 成员引用

kotlin和Java8一样，如果把函数转换成一个值，你就可以传递他。使用::元素安抚来转换。

```text
val getAge = Person::age
```

这种表达式称为`成员引用`，它提供了简明语法，来创建一个调用单个方法或者访问单个属性的函数值。

成员引用和调用该函数的lambda具有一样的类型，可以互换使用：

```kotlin
people.maxBy ( Person::age )
```

还可以引用顶层函数

```kotlin
fun salute() = println("Salute!")
run(::salute)
```

可以用`构造方法引用`存储或者延期执行创建类实例的动作。构造方法引用的形式是在双冒号后执行类名称：

```kotlin
val createPerson = ::Person //创建Person实例的动作被保存成了值
val p = createPerson("Alice",29)
println(p)
```

还可以用同样的方式引用扩展函数：

```kotlin
fun Person.isAdult() = age >= 18
val predicate = Person::isAdult
```

## 5.2 集合的函数式API

### 5.2.1 基础：filter和map

filter和map函数形成了集合操作的基础，很多集合操作都是借助它们来表达的。

```kotlin
val list = listOf(1, 2, 3, 4)
println(list.filter { it % 2 == 0 }) //[2, 4]
```

```kotlin
val list = listOf(1, 2, 3, 4)
println(list.map { it * it }) //[1, 4, 9, 16]
```

### 5.2.2 “all” “any” “count” 和 “find”：对集合应用判断式

all函数判断是否所有元素都满足判断式。

any函数判断是否至少存在一个匹配的元素。

count函数返回满足判断式元素的个数。

find函数找到满足判断的元素。

```kotlin
println(people.all(canBeInClude27)) //false
println(people.any(canBeInClude27)) //true
println(people.count(canBeInClude27)) //1
println(people.find(canBeInClude27)) //Person(name=Alice, age=27)
```

### 5.2.3 groupBy：把列表转换成分组的map

```kotlin
val people = listOf(Person("Alice", 27), Person("Bob", 31), Person("Carol", 31))
println(people.groupBy { it.age })
//{27=[Person(name=Alice, age=27)], 31=[Person(name=Bob, age=31), Person(name=Carol, age=31)]}
```

### 5.2.4 flatMap和flatten：处理嵌套集合中的元素

flatMap函数做了两件事情：首先根据作为实参给定的函数对集合中的每个元素做变换，然后把多个列表合并成一个列表。

```kotlin
val strings = listOf("abc","def")
println(strings.flatMap(String::toList))
```

```kotlin
val books = listOf(Book("Thursday Next", listOf("Jasper Fforde")),
                   Book("Mort", listOf("Terry Pratchett")),
                   Book("Good Omens", listOf("Terry Pratchett", "Neil Gaiman")))
println(books.flatMap(Book::authors).toSet()) // toSet去重
//[Jasper Fforde, Terry Pratchett, Neil Gaiman]
```

## 5.3 惰性集合操作：序列

```kotlin
people.map(Person::name).filter{it.startsWith("A")}
```

Kotlin标准库参考文档有说明，filter和map都会返回一个列表。这意味着上面例子中的链式调用会创建两个列表：一个保存filter函数的结果，另一个保存map函数的结果。如果源列表只有两个元素，这不是什么问题，但是如果有一百万个元素，调用就会变得十分低效。

为了提高效率，可以把操作变成使用序列，而不是直接使用。

```kotlin
    people.asSequence()
            .map(Person::name)
            .filter{it.startsWith("A")}
            .toList()
```

Sequence接口的强大之处在于其操作的实现方式。序列中的元素求值是惰性的。因此可以使用序列更高效地对集合元素执行链式操作，而不需要创建额外的集合来保存过程中产生的中间结果。

可以调用扩展函数asSequence把任意集合转换成序列，调用toList来做反向的转换。

### 5.3.1 执行序列操作：中间和末端操作

中间操作始终是惰性的。

```kotlin
    listOf(1, 2, 3, 4)
            .asSequence()
            .map { print("map($it) ");it * it }
            .filter { print("filter($it) ");it % 2 == 0 }
```

执行这段代码并不会在控制台上输出任何内容。这意味着map和filter变换被延期了，它们只有在获取结果的时候才会被应用。

```kotlin
    listOf(1, 2, 3, 4)
            .asSequence()
            .map { print("map($it) ");it * it }
            .filter { print("filter($it) ");it % 2 == 0 }
            .toList()
    //map(1) filter(1) map(2) filter(4) map(3) filter(9) map(4) filter(16)
```

末端操作触发执行了所有的延期计算。

```kotlin
println(listOf(1, 2, 3, 4)
        .asSequence()
        .map { it * it }
        .find { it > 3 })
```

 

![&#x53CA;&#x65E9;&#x6C42;&#x503C;&#x5728;&#x6574;&#x4E2A;&#x96C6;&#x5408;&#x4E0A;&#x6267;&#x884C;&#x6BCF;&#x4E2A;&#x64CD;&#x4F5C;&#xFF1B;&#x60F0;&#x6027;&#x6C42;&#x503C;&#x5219;&#x9010;&#x4E2A;&#x5904;&#x7406;&#x5143;&#x7D20;](../.gitbook/assets/image%20%283%29.png)

### 5.3.2 创建序列

{% code-tabs %}
{% code-tabs-item title="使用generateSequence函数创建序列" %}
```kotlin
val naturalNumbers = generateSequence(0) { it + 1 }
val numbersTo100 = naturalNumbers.takeWhile { it <= 100 }
println(numbersTo100.sum())
```
{% endcode-tabs-item %}
{% endcode-tabs %}

## 5.4 使用Java函数式接口

### 5.4.1 把lambda当做参数传递给Java方法

### 5.4.2 SAM构造方法：显式地把lambda转换成函数式接口

## 5.5 带接收者的lambda：“with”与“apply”

### 5.5.1 “with”函数

```kotlin
fun alphabet(): String {
    val result = StringBuilder()
    for (letter in 'A'..'Z'){
        result.append(letter)
    }
    result.append("\nNow I know the alphabet!")
    return result.toString()
}
```

调用result实例上好几个不同的方法，而且每次调用都要重复result这个名称。

```kotlin
fun alphabet(): String {
    val result = StringBuilder()
    return with(result){
        for(letter in 'A'..'Z'){
            append(letter)
        }
        append("\nNow I know the alphabet!")
        toString()
    }
}
```

进一步重构alphabet函数，去掉额外的StringBuilder变量。

```kotlin
fun alphabet() = with(StringBuffer()) {
    for (letter in 'A'..'Z') {
        append(letter)
    }
    append("\nNow I know the alphabet!")
    toString()
}
```

### 5.5.2 “apply”函数

apply函数几乎和with函数一模一样，唯一的区别是apply始终会返回作为实参传递给它的对象。

```kotlin
fun alphabet() = StringBuilder().apply {
    for (letter in 'A'..'Z') {
        append(letter)
    }
    append("\nNow I know the alphabet")
}.toString()
```

{% code-tabs %}
{% code-tabs-item title="使用apply初始化一个TextView" %}
```kotlin
fun createViewWithCustomAttributes(context:Context) = 
    TextView(context).apply{
        text = "Sample Text"
        textSize = 20.0
        setPadding(10,0,0,0)
    }
```
{% endcode-tabs-item %}
{% endcode-tabs %}

标准库函数buildString负责创建StringBuilder并调用toString。buildString的实参是一个带接收者的lambda，接收者就是StringBuilder。

```kotlin
fun alphabet() =  buildString{
    for (letter in 'A'..'Z') {
        append(letter)
    }
    append("\nNow I know the alphabet")
}
```

