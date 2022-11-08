---
title: "Android-Gson"
date: "2022-11-07"
categories:
    - "Android"
tags:
    - "Android"
    - "Android框架"
toc: true
indent: false
original: true
draft: false
---

### 什么是Gson

Gson 是一个Java库，可用于将Java对象转换为它们的Json 表示形式。它还可以用于将Json 字符串转换为等效的Java对象。Json 的解析和生成的方式很多，在 Android 平台上最常用的类库有 Gson 和 FastJson 两种，这里要介绍的是 Gson。

Gson 的 GitHub主页点击这里：[Gson](https://github.com/google/gson)

### 如何下载

Gradle:

```gradle
dependencies {
  implementation 'com.google.code.gson:gson:2.8.9'
}
```

### Json的生成

```kotlin
Gson().toJson(any)
```

### Json的解析

+ 类

```kotlin
val user : User = Gson().fromJson(json,User::class)
```

+ 复杂类型

```kotlin
val map : Map<String , Any> = Gson().fromJson(json,object : TypeToken<Map<String , Any>>() {}.type)
```

### Json的注解

#### 1.自定义字段的名字

```kotlin
@SerializedName("w")
var width:Int? = null
```

这个注解可以用来自定义序列化和反序列化过程中字段的名字。
以上面为例，当序列化的时候，会把Java bean中的字段`width`存储成`w`，在反序列化的时候会把Json的`w`这个key反序列化到Java bean的`width`字段上。

#### 2.定义那些字段需要被序列化或者反序列化

**注意，在Java中，所有用transient声明的字段，都不会被Gson序列化和反序列化**

Gson提供了注解来分别的控制某一个字段是否需要被序列化或者反序列化。对序列化和反序列化分开控制。

使用@Expose注解，我们可以对序列化和反序列化单独控制，该注解有两个值，分别是deserialize和serialize，比如如下的例子

```kotlin
class Account {
    
    // 声明该字段不参与反序列化
    @Expose(deserialize = false)
    var accountNumber : String? = null
    
    // 只要有一个字段使用了Expose注解，所有需要参与序列化和反序列化的字段都要有这个注解
    // 因为这个注解要么不生效，如果生效的话，就只会对有Expose注解的字段进行处理。
    @Expose
    var iban : String? = null
    
    // 声明该字段不参与序列化
    @Expose(serialize = false)
    var owner : String? = null
    
    // 声明该字段序列化和反序列化都不参与
    @Expose(serialize = false, deserialize = false)
    var address : String? = null
    
    var pin : String? = null
    
}
```

要使该注解生效，必须对Gson进行配置，如下

```kotlin
val builder : GsonBuilder = GsonBuilder()
builder.excludeFieldsWithoutExposeAnnotation()
val gson : Gson = builder.create()
```

如果我们不对Gson进行配置的话，该注解就不会生效，这样就会默认所有的字段都会被序列化和反序列化。

通过对Gson进行配置，只有带有`Expose`注解的字段才会被Gson进行序列化或者反序列化。

#### 3.对Java Bean进行版本控制，这个使用的很少，比如

```kotlin
class SoccerPlayer {
    
    var name : String? = null
    
    //表明这个属性是1.2版本之后才加入的
    @Since(1.2)
    var shirtNumber : Int? = null
    
    //表明这个属性是在0.9版本上已经被移出了
    @Until(0.9)
    var country : String? = null
    
    var teamName : String? = null
    
}
```

和`Expose`一样，要使用这两个注解，也需要对Gson进行配置，如下

```kotlin
val builder : GsonBuilder = GsonBuilder()
//在这里，我们定义版本是1.0，由于shirtNumber在1.2才加入，所以不生效
//country在0.9被移除，所以也不生效
builder.setVersion(1.0)
val gson : Gson = builder.create()
```