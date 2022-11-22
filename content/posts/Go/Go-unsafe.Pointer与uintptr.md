---
title: "Go-unsafe.Pointer与uintptr"
date: "2022-11-21"
categories:
    - "Go"
tags:
    - "Go"
toc: true
indent: false
original: true
draft: false
---

# 一、查看普通指针类型

```go
type Admin struct {
	Name string
	Age  int
}

func main() {
	var a *int64
	var b *bool
	var c int32 = 64
	admin1 := Admin{
		Name: "seekload",
		Age:  18,
	}
	p1 := &admin1
	admin2 := &Admin{
		Name: "seekload",
		Age:  18,
	}
	p2 := &admin2
	fmt.Printf("%T\n", a)           //输出:*int64
	fmt.Printf("%T\n", b)           //输出:*bool
	fmt.Printf("%T\n", &c)          //输出*int32
	fmt.Printf("%T\n", admin1)      //输出:main.Admin
	fmt.Printf("%T\n", p1)          //输出:*main.Admin
	fmt.Printf("%T\n", admin2)      //输出:*main.Admin
	fmt.Printf("%T\n", p2)          //输出:**main.Admin
	fmt.Println(reflect.TypeOf(p2)) //输出:**main.Admin
	
	fmt.Println(reflect.TypeOf(Admin{Name: "seekload", Age: 18,})) //输出:main.Admin
	fmt.Println(reflect.TypeOf(Admin{}))                           //输出:main.Admin
}
```

# 二、指针变量类型不能转换

Go 是强类型语言，声明变量之后，变量的类型是不可以改变的；

一般将 *T 看作指针类型，表示一个指向 T 类型变量的指针，不同类型的指针也不允许相互转化

## （1）

```go
var i int32 = 30
i = int64(i) // 报错
```

## （2）

此类情况比较特殊

```go
var i int64 = 30
fmt.Println(reflect.Typeof(i)) // 输出 int64
i2 := int32(i)
fmt.Println(reflect.Typeof(i2)) // 输出 int32
```

## （3）

```go
var i int64 = 30
iPtr1 := &i
iPtr1 = (*int32)(iPtr1) // 报错
```

## （4）


```go
var i int64 = 30
iPtr1 := &i
iPtr2 := (*int32)(iPtr1) // 报错
```

# 三、[unsafe](https://so.csdn.net/so/search?q=unsafe&spm=1001.2101.3001.7020).Pointer类型可指针类型转换

## 1.介绍

unsafe.Pointer不是函数，是一个类型；
unsafe.Pointer 通用指针类型，一种特殊类型的指针，可以包含任意类型的地址，能实现不同的指针类型之间进行转换；
官方文档里还描述了 Pointer 的四种操作规则
(1)任何类型的指针都可以转化成 unsafe.Pointer
(2)unsafe.Pointer 可以转化成任何类型的指针
(3)uintptr 可以转换为 unsafe.Pointer
(4)unsafe.Pointer 可以转换为 uintptr

```go
// unsafe.go
type ArbitraryType int
```

```go
// unsafe.go
type Pointer *ArbitraryType
```

## 2.任何类型的指针都可以转化成 unsafe.Pointer

```go 
var i int32 = 30
iPtr1 := &i
fmt.Println(reflect.TypeOf(iPtr1)) // 输出:*int32
var iPtr2 unsafe.Pointer = unsafe.Pointer(iPtr1)
fmt.Println(iPtr2)                 // 输出:0xc0000aa058
fmt.Println(reflect.TypeOf(iPtr2)) // 输出:unsafe.Pointer
```

```go
var i int32 = 30
iPtr1 := &i
fmt.Println(reflect.TypeOf(iPtr1)) // 输出:*int32
iPtr2 := unsafe.Pointer(iPtr1)
fmt.Println(iPtr2)                 // 输出:0xc000016098
fmt.Println(reflect.TypeOf(iPtr2)) // 输出:unsafe.Pointer
```

> 注：不能下面这样使用

```go
var i int32 = 30
iPtr1 := &i
fmt.Println(reflect.TypeOf(iPtr1)) // 输出:*int32
iPtr1 = unsafe.Pointer(iPtr1) // 错误
```

## 3.不同类型的指针允许相互转化

不同类型的指针允许相互转化实际上是运用了第 1、2 条规则，我们就着例子看下：

```go
var i int32 = 30
iPtr1 := &i
fmt.Println(reflect.TypeOf(iPtr1)) // 输出:*int32
var iPtr2 *int64 = (*int64)(unsafe.Pointer(Ptr1))
fmt.Println(reflect.TypeOf(iPtr2)) // 输出:*int64
```

```go
var i int32 = 30
iPtr1 := &i
fmt.Println(*iPtr1)	// 输出:30

*iPtr1 = 25
fmt.Println(*iPtr1, i)	// 输出:25 25

var iPtr2 *int64 = (*int64)(unsafe.Pointer(iPtr1))
fmt.Println(*iPtr2)	// 输出:25
*iPtr2 = 20
fmt.Println(*iPtr2, *iPtr1, i)	// 输出:20  20  20 
```

> 注：不能下面这样使用

```go
var i int32 = 30
iPtr1 := &i
fmt.Println(reflect.TypeOf(iPtr1)) // 输出:*int32
iPtr1 = (*int64)(unsafe.Pointer(iPtr1)) // 错误
```

# 四、内置类型unitptr

## 1.介绍

uintptr 是 Go 内置类型，表示无符号整数，可存储一个完整的地址。常用于指针运算，只需将 unsafe.Pointer 类型转换成 uintptr 类型，做完加减法后，转换成 unsafe.Pointer，通过 * 操作，取值或者修改值都可以。

```go
// builtin.go
type uintptr uintptr
```

```go
var a uintptr = 4
fmt.Println(a) // 输出4
```

## 2.通过指针偏移修改结构体成员

```go
admin := Admin{
		Name: "seekload",
		Age:  18,
	}
ptr := &admin
fmt.Println(*ptr) //输出:{seekload 18}
//这样的方式的话,获取的是第一个字段的指针
name := (*string)(unsafe.Pointer(ptr))
fmt.Println(name)
*name = "四哥"
fmt.Println(*ptr) //输出:{四哥 18}
/* 成员变量 Age 不是第一个字段，想要修改它的值就需要内存偏移。我们先将 admin 的指针转化为 uintptr，
再通过 unsafe.Offsetof() 获取到 Age 的偏移量，两者都是 uintptr，进行相加指针运算获取到成员 Age 的地址，
最后需要将 uintptr 转化为 unsafe.Pointer，再转化为 *int，才能对 Age 操作。*/
age := (*int)(unsafe.Pointer(uintptr(unsafe.Pointer(ptr)) + unsafe.Offsetof(ptr.Age))) // 2
fmt.Println(reflect.TypeOf(age))                                               //输出:*int
*age = 35
fmt.Println(*ptr) //输出:{四哥 35}
```

```go
// unsafe.go
func Offsetof(x ArbitraryType) uint
```

# 总结

这篇文章我们简单介绍了普通指针类型、unsafe.Pointer 和 uintptr 之间的关系，记住三点：

### (1)unsafe.Pointer 可以实现不同类型指针之间相互转化；
### (2)uintptr 搭配着 unsafe.Pointer 使用，实现指针运算；
### (3)不过，官方不推荐使用 unsafe 包，正如它的命名一样，是不安全的，比如涉及到内存操作，这是绕过 Go 本身设计的安全机制的，不当的操作，可能会破坏一块内存，而且这种问题非常不好定位。
