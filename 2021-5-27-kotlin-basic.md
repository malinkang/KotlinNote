---
title: kotlin基础
date: 2018-08-25 12:26:55
tags: ["Kotlin"]
cover: https://malinkang-1253444926.cos.ap-beijing.myqcloud.com/blog/images/cover/千与千寻12.png
---

## 2.1 基础要素：函数和变量

### 2.1.1 Hello,world!

```kotlin
fun main(args: Array<String>) {
    println("Hello, world!")
}
```

### 2.1.2 函数

```kotlin
fun main(args: Array<String>) {
    println(max(1,2))
}

fun max(a:Int,b:Int):Int{
    return if(a>b) a else b
}
```

#### 表达式函数体

函数体由单个表达式构成的，可以用这个表达式作为完整的函数体，并去掉花括号和`return`语句：

```kotlin
fun max(a:Int,b:Int):Int = if(a>b) a else b
```

如果函数体写在花括号中，我们说这个函数有`代码块体（block body)`。如果它直接返回一个表达式，它就有`表达式体（expression body）`。

可以进一步简化`max`函数，省掉返回类型：

```kotlin
fun max(a:Int,b:Int) = if(a>b) a else b
```

### 2.1.3 变量

#### 可变变量和不可变量

val（来自value）不可变引用。使用val声明的变量不能在初始化之后再次赋值。

```kotlin
class VariablesDemo {

    var a = 1
    val b = 2

    @JvmField //不能修饰private
    val c = 3


    companion object {
        var d = 4
        @JvmField
        val e = 5
        val f = 6
        const val g = 7
        private const val h = 8
    }
}
```
反编译

```java
public final class VariablesDemo {
   private int a = 1; //set get 略
   private final int b = 2; //只有get方法 略
   @JvmField 
   public final int c = 3; //没有set get方法 
   private static int d = 4; //有set get方法 略
   @JvmField
   public static final int e = 5; 
   private static final int f = 6; //const修饰的是public 没有则是private
   public static final int g = 7;
   private static final int h = 8; //用private修饰的
}
```
### 2.1.4  更简单的字符串格式化：字符串模版

## 2.2 类和属性

### 2.2.1 属性

### 2.2.2 自定义访问器

### 2.2.3 Kotlin源码布局：目录和包

## 2.3 表示和处理选择：枚举和“when”

### 2.3.1 声明枚举类

```java
public enum Color {

    RED(255, 0, 0), ORANGE(255, 165, 0), YELLOW(255, 255, 0),
    GREEN(0, 255, 0), BLUE(0, 0, 255), INDIGO(75, 0, 130), VIOLET(238, 130, 238);
    private int r;
    private int g;
    private int b;

    private Color(int r, int g, int b) {
        this.r = r;
        this.g = g;
        this.b = b;
    }

    private int rgb() {
        return (r * 256 + g) * 256 + b;
    }
}
```

```kotlin
enum class Color constructor(private val r: Int, private val g: Int, private val b: Int) {

    RED(255, 0, 0), ORANGE(255, 165, 0), YELLOW(255, 255, 0),
    GREEN(0, 255, 0), BLUE(0, 0, 255), INDIGO(75, 0, 130), VIOLET(238, 130, 238);

    private fun rgb(): Int {
        return (r * 256 + g) * 256 + b
    }
}
```

### 2.3.2 使用“when”处理枚举类

```kotlin
fun getMnemonic(color: Color) =
        when (color) {
            Color.RED -> "Richard"
            Color.ORANGE -> "Of"
            Color.YELLOW -> "York"
            Color.GREEN -> "Gave"
            Color.BLUE -> "Battle"
            Color.INDIGO -> "In"
            Color.VIOLET -> "Vain"
        }

println(getMnemonic(Color.BLUE)) //Battle
```

```kotlin
fun getWarmth(color: Color) = when (color) {
    Color.RED, Color.ORANGE, Color.YELLOW -> "warm"
    Color.GREEN -> "neutral"
    Color.BLUE, Color.INDIGO, Color.VIOLET -> "cold"
}
println(getWarmth(Color.ORANGE)) //Warm
```

### 2.3.3 在“when”结构上使用任意对象

Kotlin中的when结构比Java中的Switch强大得多。Switch要求必须使用常量作为分支条件，when允许使用任何对象。

```kotlin
fun mix(c1: Color, c2: Color) =
        when (setOf(c1, c2)) {
            setOf(Color.RED, Color.YELLOW) -> Color.ORANGE
            setOf(Color.YELLOW, Color.BLUE) -> Color.GREEN
            setOf(Color.BLUE, Color.VIOLET) -> Color.INDIGO
            else -> throw Exception("Dirty color")
        }
```

### 2.3.4 使用不带参数的“when”

```kotlin
fun mixOptimized(c1: Color, c2: Color) =
        when {
            (c1 == Color.RED && c2 == Color.YELLOW) ||
                    (c1 == Color.YELLOW && c2 == Color.RED) -> Color.ORANGE
            (c1 == Color.YELLOW && c2 == Color.BLUE) ||
                    (c1 == Color.BLUE && c2 == Color.YELLOW) -> Color.GREEN
            (c1 == Color.BLUE && c2 == Color.VIOLET) ||
                    (c1 == Color.VIOLET && c2 == Color.BLUE) -> Color.INDIGO
            else -> throw Exception("Dirty color")
        }
```

### 2.3.5 智能转换：合并类型检查和转换

### 2.3.6 重构：用“when”替代“if”

### 2.3.7 代码块作为“if”和“when”的分支

## 2.4 迭代事物：“while”循环和“for”循环

### 2.4.1 “while”循环

### 2.4.2 迭代数字：区间和数列

```kotlin
for (i in 0..10) {
    print("$i ") //0 1 2 3 4 5 6 7 8 9 10 
}
for(i in 0..10 step 2){
    print("$i ") //0 2 4 6 8 10
}
//递减
for (i in 10 downTo 5) {
    print("$i ") //10 9 8 7 6 5 
}
for (i in 10 downTo 5 step 2) {
    print("$i ") //10 8 6 
}
//不包含
for (i in 1 until 10) {
    print("$i ") //1 2 3 4 5 6 7 8 9
}
```

### 2.4.3 迭代map

### 2.4.4 使用“in”检查集合和区间的成员

## 2.5 Kotlin中的异常

### 2.5.1 “try” “catch”和“finally”

### 2.5.2 “try”作为表达式

## 参考
* [控制流](https://www.kotlincn.net/docs/reference/control-flow.html)
* [返回和跳转](https://www.kotlincn.net/docs/reference/returns.html)