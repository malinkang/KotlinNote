---
title: Kotlin实现单例
date: 2018-08-28T12:26:55.000Z
tags:
  - Kotlin
link: https://www.notion.so/2021-5-27-kotlin-singletone-9dd179c6cf4a42958b769e7c0035353f
notionID: 9dd179c6-cf4a-4295-8b76-9e7c0035353f
---

## 饿汉式

```java
//java实现
public class Singleton {
    private static Singleton instance = new Singleton();
    private Singleton(){}
    public static Singleton getInstance(){
        return instance;
    }
}
```

```kotlin
//kotlin实现
object Singleton
```

```java
//反编译上面的代码 Tools->Kotlin->Show Kotlin Bytecode
public final class Singleton {
   @NotNull
   public static final Singleton INSTANCE;

   private Singleton() {
   }

   static {
      Singleton var0 = new Singleton();
      INSTANCE = var0;
   }
}
```

## 懒汉式

```java
//java实现
public class Singleton {
    private static Singleton instance ;
    private Singleton(){}
    public static Singleton getInstance(){
        if(instance==null) {
            new Singleton();
        }
        return instance;
    }
}
```

```kotlin
class Singleton private constructor() {
  companion object {
    private var instance: Singleton? = null
      get() {
        if (field == null) {
          field = Singleton()
        }
        return field
      }
    fun get(): Singleton {
      return instance!!
    }
  }
}
```

## 双重校验锁

```java
//Java实现
public class Singleton {
    private static Singleton instance ;
    private Singleton(){}
    public static Singleton getInstance(){
        if(instance==null) {
          synchronized (Singleton.class){
              if(instance==null){
                  instance = new Singleton();
              }
          }
        }
        return instance;
    }
}
```

```kotlin
//kotlin实现 不过这种没法传入参数
class Singleton private constructor() {
  companion object {
    val instance by lazy {
      Singleton()
    }
  }
}
```

```kotlin
//参考https://developer.android.com/codelabs/android-room-with-a-view-kotlin#7
//允许传入参数的DCL
class Singleton private constructor(a:Int) {
  companion object {
    @Volatile
    private var INSTANCE:Singleton? = null
    fun get(a:Int):Singleton{
      return INSTANCE?: synchronized(this){
        val instance = Singleton(a)
        INSTANCE = instance
        instance
      }
    }
  }
}
```

## 参考

* [Kotlin下的5种单例模式](https://juejin.cn/post/6844903590545326088)
* [Kotlin singletons with argument](https://bladecoder.medium.com/kotlin-singletons-with-argument-194ef06edd9e)
* [单例](https://www.liaoxuefeng.com/wiki/1252599548343744/1281319214514210)