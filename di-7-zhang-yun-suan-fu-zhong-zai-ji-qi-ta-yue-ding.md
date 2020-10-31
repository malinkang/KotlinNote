# 第7章 运算符重载及其他约定

## 7.1 重载算术运算符

### 7.1.1 重载二元算术运算

```kotlin
data class Point(val x: Int, val y: Int) {
    operator fun plus(other: Point): Point {
        return Point(x + other.x, y + other.y)
    }
}
val p1 = Point(10, 20)
val p2 = Point(30, 40)
println(p1 + p2) //Point(x=40, y=60)
```

```kotlin
operator fun Point.plus(other: Point): Point {
    return Point(x + other.x, y + other.y)
}
```

| 表达式 | 函数名 |
| :--- | :--- |
| a \* b | times |
| a / b | div |
| a % b | mod |
| a + b | plus |
| a - b | minus |

```kotlin
operator fun Point.times(scale: Double): Point {
    return Point((x * scale).toInt(), (y * scale).toInt())
}
val p = Point(10, 20)
print(p * 1.5) //Point(x=15, y=30)
```

Kotlin运算符不会自动支持交换性。如果希望用户能够使用15\*p以外，还能使用p\*1.6，你需要为它定义个单独的运算符：

```kotlin
operator fun Double.times(p:Point):Point{
    return Point((p.x * this).toInt(), (p.y * this).toInt())
}
```

```text
operator fun Char.times(count: Int): String {
    return toString().repeat(count)
}
println('a' * 3) //aaa
```

### 7.1.2 重载符合运算符

### 7.1.3 重载一元运算符

```kotlin
operator fun Point.unaryMinus():Point{
    return Point(-x,-y)
}
val p = Point(10,20)
println(-p)
```

| 表达式 | 函数名 |
| :---: | :---: |
| +a | unaryPlus |
| -a | unaryMinus |
| !a | not |
| ++a，a++ | inc |
| --a，a-- | dec |

```kotlin
operator fun BigDecimal.inc() = this +BigDecimal.ONE
var bd = BigDecimal.ZERO
println(bd++) //0
println(++bd) //2
```

## 7.2 重载比较运算符

### 7.2.1 等号运算符：“equals”

```kotlin
data class Point(val x: Int, val y: Int) {
    override fun equals(other: Any?): Boolean {
    //这里使用了恒等运算符===来检查参数与调用equals的对象是否相同。恒等运算符与Java中的==运算符
    //是完全相同的；===运算符不能被重载
        if (other === this) return true
        if (other !is Point) return false
        return other.x == x && other.y == y
    }
}
println(Point(10,20)==Point(10,20)) //true
println(Point(10,20)!=Point(5,5)) //true
println(null == Point(1,2)) //false
```

### 7.2.2 排序运算符：compareTo

```kotlin
class Person(val firstName: String, val lastName: String) : Comparable<Person> {
    override fun compareTo(other: Person): Int {
        return compareValuesBy(this, other, Person::lastName, Person::firstName)
    }
}
```

## 7.3 集合与区间的约定

### 7.3.1 通过下标来访问元素：“get”和“set”

```kotlin
operator fun Point.get(index: Int): Int {
    return when (index) {
        0 -> x
        1 -> y
        else ->
            throw IndexOutOfBoundsException("Invalid coordinate $index")
    }
}
val p = Point(10,20)
println(p[1]) //20
```

```kotlin
operator fun MutablePoint.set(index: Int, value: Int) {
    when (index) {
        0 -> x = value
        1 -> y = value
        else ->
            throw IndexOutOfBoundsException("Invalid coordinate $index")

    }
}
val p = MutablePoint(10, 20)
p[1] = 42
println(p) //MutablePoint(x=10, y=42)
```

### 7.3.2 “in”的约定

in运算符，用于检查某个对象是否属于集合。相应的函数叫做`contains`。

```kotlin
data class Rectangle(val upperLeft:Point,val lowerRight:Point)

operator fun Rectangle.contains(p:Point):Boolean{
    return p.x in upperLeft.x until lowerRight.x &&
            p.y in upperLeft.y until lowerRight.y
}
val rect = Rectangle(Point(10,20),Point(50,50))
println(Point(20,30) in rect) //true
println(Point(5,5) in rect) //false
```

### 7.3.3 rangeTo的约定

```kotlin
val now = LocalDate.now()
val vacation = now..now.plusDays(10)
println(now.plusWeeks(1) in vacation) //true
val n = 9
println(0..(n+1)) //0..10
(0..n).forEach(::print) //0123456789
```

### 7.3.4 在“for”循环中使用“iterator”的约定

```kotlin
operator fun ClosedRange<LocalDate>.iterator(): Iterator<LocalDate> =
        object : Iterator<LocalDate> {
            var current = start
            override fun hasNext() = current <= endInclusive
            override fun next() = current.apply {
                current = plusDays(1)
            }
        }
val newYear = LocalDate.ofYearDay(2017, 1)
val daysOff = newYear.minusDays(1)..newYear
for (dayOff in daysOff) {
    println(dayOff)
}
//    2016-12-31
//    2017-01-01
```

