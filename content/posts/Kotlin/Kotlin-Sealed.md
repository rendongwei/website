---
title: "Kotlin-Sealed密封类"
date: "2022-07-08"
categories:
    - "Kotlin"
tags:
    - "Kotlin"
toc: true
indent: false
original: true
draft: false
---
# 前言
在代码中，我们经常需要限定一些有限集合的状态值，例如：

+ 网络请求：成功——失败
+ 账户状态：VIP——穷逼VIP——普通
+ 工具栏：展开——半折叠——收缩

等等。

通常情况下，我们会使用`enum class`来做封装，将可见的状态值通过枚举来使用。

```kotlin
enum class NetworkState(val value: Int) {
    SUCCESS(0),
    ERROR(1)
}
```

但枚举的缺点也很明显，首先，枚举比普通代码更占内存，同时，每个枚举只能定义一个实例，不能拓展更多信息。

除此之外，还有种方式，通过抽象类来对状态进行封装，但这种方式的缺点也很明显，它打破了枚举的限制性，所以，Kotlin给出了新的解决方案——Sealed Class（密封类）。

## 1.创建状态集

下面我们以网络请求的例子来看下具体如何使用Sealed Class来进行状态的封装。

和抽象类类似，Sealed Class可用于表示层级关系。它的子类可以是任意的类：data class、普通Kotlin对象、普通的类，甚至也可以是另一个密封类，所以，我们定义一个Result Sealed Class：

```kotlin
sealed class Result<out T : Any> {
    data class Success<out T : Any>(val data: T) : Result<T>()
    data class Error(val exception: Exception) : Result<Nothing>()
}
```

当然，也不一定非要写在顶层类中：

```kotlin
sealed class Result<out T : Any> 
data class Success<out T : Any>(val data: T) : Result<T>()
data class Error(val exception: Exception) : Result<Nothing>()
```

这样也是可以的，它们的区别在于引用的时候，是否包含顶层类来引用而已。

大部分场景下，还是建议第一种方式，可以比较清晰的展示调用的层级关系。

在这个例子中，我们定义了两个场景，分别是Success和Error，它表示我们假设的网络状态就这两种，分别在每种状态下，例如Success，都可以传入自定义的数据类型，因为它本身就是一个class，所以借助这一点，就可以自定义状态携带的场景值。在上面这个例子中，我们定义在Success中，传递data，而在Error时，传递Exception信息。

所以，使用Sealed Class的第一步，就是对场景进行封装，梳理具体的场景枚举，并定义需要传递的数据类型。

> 如果场景值不需要传递数据，那么可以简单的使用：object xxxx，定义一个变量即可。

## 2.使用

接下来，我们来看下如何使用Sealed Class。

```kotlin
fun main() {
    // 模拟封装枚举的产生
    val result = if (true) {
        Result.Success("Success")
    } else {
        Result.Error(Exception("error"))
    }

    when (result) {
        is Result.Success -> print(result.data)
        is Result.Error -> print(result.exception)
    }
}
```

大部分场景下，Sealed Class都会配合when一起使用，同时，如果when的参数是Sealed Class，在IDE中可以快速补全所有分支，而且不会需要你单独补充else 分支，因为Sealed Class已经是完备的了。

所以when和Sealed Class真是天作之合。

## 3.进一步简化

其实我们还可以进一步简化代码的调用，因为我们每次使用Sealed Class的时候，都需要when一下，有些时候，也会产生一些代码冗余，所以，借助拓展函数，我们进一步对代码进行简化。

```kotlin 
inline fun Result<Any>.doSuccess(success: (Any) -> Unit) {
    if (this is Result.Success) {
        success(data)
    }
}

inline fun Result<Any>.doError(error: (Exception?) -> Unit) {
    if (this is Result.Error) {
        error(exception)
    }
}
```

这里我对Result进行了拓展，增加了doSuccess和doError两个拓展，同时接收两个高阶函数来接收处理行为，这样我们在调用的时候就更加简单了。

```kotlin
result.doSuccess { }
result.doError { }
```

所以when和Sealed Class和拓展函数，真是天作之合。

那么你一定好奇了，Sealed Class又是怎么实现的，其实反编译一下就一目了然了，实际上Sealed Class也是通过抽象类来实现的，编译器生成了一个只能编译器调用的构造函数，从而避免其它类进行修改，实现了Sealed Class的有限性。

## 4.封装

Sealed Class与抽象类类似，可以对逻辑进行拓展，我们来看下面这个例子。

```kotlin
sealed class TTS {
    
    abstract fun speak()

    class BaiduTTS(val value: String) : TTS() {
        override fun speak() = print(value)
    }

    class TencentTTS(val value: String) : TTS() {
        override fun speak() = print(value)
    }
}
```

这时候如果要进行拓展，就很方便了，代码如下所示。

```kotlin
class XunFeiTTS(val value: String) : TTS() {
    override fun speak() = print(value)
}
```

所以，Sealed Class可以说是在抽象类的基础上，增加了对状态有限性的控制，拓展与抽象，比枚举更加灵活和方便了。

再例如前面网络的封装：

```kotlin
sealed class Result<out T : Any> {
    data class Success<out T : Any>(val data: T) : Result<T>()
    sealed class Error(val exception: Exception) : Result<Nothing>() {
        class RecoverableError(exception: Exception) : Error(exception)
        class NonRecoverableError(exception: Exception) : Error(exception)
    }

    object InProgress : Result<Nothing>()
}
```

通过Sealed Class可以很方便的对Error类型进行拓展，同时，增加新的状态也非常简单，更重要的是，通过IDE的自动补全功能，IDE可以自动生成各个条件分支，避免人工编码的遗漏。
