---
title: "Go-空结构体的应用与实现原理"
date: "2022-11-17"
categories:
    - "Go"
tags:
    - "Go"
toc: true
indent: false
original: true
draft: false
---

# 空结构体的应用与实现原理

## 一、空结构体介绍

首先来看看空结构体是什么；空结构体也是结构体类型，具有结构体的一切特性，但该结构体中没有任何字段组合。

```go
type a struct {
}
func main() {
	fmt.Println(a{}) //输出{}
}
```

```go
type a struct {
}

func main() {
	fmt.Println(reflect.TypeOf(a{}))  //输出:main.a  注意:不能直接写a,必须得写a{}
	fmt.Println(reflect.TypeOf(&a{})) //输出:*main.a
	b := a{}
	p1 := &b
	p2 := *p1
	fmt.Println(reflect.TypeOf(b))  //输出main.a
	fmt.Println(reflect.TypeOf(p1)) //输出:*main.a
	fmt.Println(reflect.TypeOf(p2)) //输出:main.a

	c := &a{}
	p3 := &c
	p4 := *c
	fmt.Println(reflect.TypeOf(c))  //输出:*main.a
	fmt.Println(reflect.TypeOf(p3)) //输出:**main.a
	fmt.Println(reflect.TypeOf(p4)) //输出:main.a

}
```

## 二.空结构体变量的占用空间

### 1. unsafe.Sizeof函数验证

struct{}的类型占用的空间是0；我们通过unsafe.Sizeof函数来验证一下；unsafe.Sizeof函数的作用是返回一个数据类型所占的空间大小，我们验证一下：

```go
var s struct{}
fmt.Println(unsafe.Sizeof(s)) // 输出： 0
```

### 2.reflect的类型验证

我们看到，通过映射变量s的类型，输出空类型的空间大小也是0：

```go
var s struct{}
typ := reflect.TypeOf(s)
fmt.Println(typ.Size()) //输出： 0
```

### 3.空结构体类型变量的地址

#### 1.所有空结构体类型的变量地址都是一样

我们知道，在编程语言中，变量的作用就是在内存中，标记和存储数据的。也就是说每个变量会对应着一块内存空间，既然是内存空间，那就应该有对应的内存地址。那空结构体类型变量的地址是什么呢？我们通过如下代码来看下：

```go
type emptyStruct struct{} 
func main() {
    a := struct{}{}
    b := struct{}{}
    c := emptyStruct{}
 
    fmt.Println(a)
    fmt.Printf("%pn", &a) //0x116be80
    fmt.Printf("%pn", &b) //0x116be80
    fmt.Printf("%pn", &c) //0x116be80
    fmt.Println(a == b) //true
}
```

发现所有空结构体类型的变量地址都是一样的

#### 2. 通过底层查看原因

在golang中，只要分配的内存为0，就返回zerobase这个变量地址；
只要你将struct{} 赋值给一个或者多个变量，它都返回这个 zerobase 的地址；
zerobase这个变量是一个 uintptr 的全局变量，占用8个字节,在go源码src/runtime/malloc.go中有定义(在runtime里多次使用到了zerobase这个变量);

```go
var zerobase uintptr
```

zerobase返回为0与go源码src/runtime/malloc.go的mallocgc函数有关系，其定义如下：

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    if gcphase == _GCmarktermination {
  throw("mallocgc called with gcphase == _GCmarktermination")
    }

    if size == 0 {
  return unsafe.Pointer(&zerobase)
    }
    ...
}
```

### 4.空结构体的应用场景

#### 1.基于map实现集合功能

一般我们用在用户不关注值内容的情况下，只是作为一个占位符来使用；
使用空结构体不占用存储空间外，还有一个语义上的原因；
一看空结构体struct{}就知道要表达的意思是不需要关心值是什么，只需要关心键值即可

```go
var CanSkipFuncs = map[string]struct{}{
		"Email":   {},
		"IP":      {},
		"Mobile":  {},
		"Tel":     {},
		"Phone":   {},
		"ZipCode": {},
	}
fmt.Println(CanSkipFuncs) //输出：map[Email:{} IP:{} Mobile:{} Phone:{} Tel:{} ZipCode:{}]
```

#### 2.与channel组合使用，实现一个信号

```go 
func (tm *simpleTokenTTLKeeper) stop() {
    select {
    case tm.stopc <- struct{}{}:
    case <-tm.donec:
    }
    <-tm.donec
}
```

#### 3.与channel组合使用,基于缓冲channel实现并发限速

```go
chan1 := make(chan struct{},1) 
go func(){
    <- chan1  //表示可以开始接收数据了，否则等待
    ...
}()
chan1 <- struct{}{}
```

