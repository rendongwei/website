---
title: "Go-并发"
date: "2022-11-11"
categories:
    - "Go"
tags:
    - "Go"
toc: true
indent: false
original: true
draft: false
---

# 前言

go并发包含两个概念：

+ goroutine:使用一个函数，但是与调用该函数是独立的。

+ channel：goroutine间通信用的管道

```go 
package main

import (
    "io"
    "os"
    "time"
)

func echo(in io.Reader, out io.Writer) {
    io.Copy(out, in)
}

func main() {
    go echo(os.Stdin, os.Stdout)
    time.Sleep(30 * time.Second)
    os.Exit(0)
}
```

使用一个goroutine在一直在后台将标准输入中的内容拷贝到标准输出当中，直到main函数退出。
使用匿名函数来新建goroutine:

```go
package main

import (
    "fmt"
    "runtime"
)

func main() {
    fmt.Println("Outside a goroutine")
    go func() {
        fmt.Println("Inside a goroutine")
    }()
    fmt.Println("Outside again")
    runtime.Gosched()
}
```

## 1.WaitGroup使用

在main函数中调用runtime.Gosched是为了挂起main本身协程，让匿名函数执行结果能打印出来。使用go关键字创建goroutine并不意味着该goroutine会马上得到执行。这个需要根据go的调度器来安排的。Gosched只能提供其他goroutine执行的机会，如果其他goroutine正在等待数据库查询或者读io的话调度器会继续执行当前的goroutine，而无法保证其他的goroutine能执行完成。要等待所有的goroutine执行完成需要调用wait group。

```go
package main

import (
    "compress/gzip"
    "io"
    "os"
)

func compress(filename string) error {
    in, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer in.Close()
    out, err := os.Create(filename + ".zip")
    if err != nil {
        return err
    }
    defer out.Close()
    
    gzout := gzip.NewWriter(out)
    _, err = io.Copy(gzout, in)
    gzout.Close()
    return err
}

func main() {
    for _, file := range os.Args {
        compress(file)
    }
}
```

该程序实现，将命令行后面提供的所有文件，压缩成对应的.zip文件。因为gzip函数是是耗io的，所以该程序在性能上没有充分利用cpu多核来实现多个文件的并行压缩：

```go 
func main() {
    var wg sync.WaitGroup
    var i int = -1
    var file string
    for i, file = range os.Args[1:] {
        wg.Add(1)
        go func(file string) {
            compress(file)
            wg.Done()
        }(file)
    }
    wg.Wait()
    fmt.Printf("Compressed %d files\n", i+1)
}
```

修改的main函数可以并发的执行多个文件压缩。main函数通过wait函数等待所有的goroutine结束才返回。注意：这里为了保证for循环每次迭代后对应的file都传入对应的goroutine当中，需要在匿名函数中添加参数file，如果直接调用compress传入file，随着for循环的迭代，前面的goroutine不一定立即得到执行导致前面的goroutine中filename和后面的一样，最后导致结果不正确。为了防止for循环当中传入到每个goroutine参数都不会被改变，必须在匿名函数当中传入参数的副本。

## 2.Mutex互斥锁使用

```go
package main

import (
    "bufio"
    "fmt"
    "os"
    "strings"
    "sync"
)

type words struct {
    sync.Mutex
    found map[string]int
}

func newWords() *words {
    return &words{found: map[string]int{}}
}

func (w *words) add(word string, n int) {
    w.Lock()
    defer w.Unlock()
    if _, ok := w.found[word]; !ok {
        w.found[word] = n
        return
    }
    w.found[word] += n
}

func tallyWords(filename string, dict *words) error {
    file, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer file.Close()
    scanner := bufio.NewScanner(file)
    scanner.Split(bufio.ScanWords)
    for scanner.Scan() {
        word := strings.ToLower(scanner.Text())
        dict.add(word, 1)
    }
    return scanner.Err()
}

func main() {
    var wg sync.WaitGroup
    w := newWords()
    for _, f := range os.Args[1:] {
        wg.Add(1)
        go func(file string) {
            if err := tallyWords(file, w); err != nil {
                fmt.Println(err.Error())
            }
            wg.Done()
        }(f)
    }
    wg.Wait()
    fmt.Println("Words that appear more than onece:")
    for word, count := range w.found {
        if count > 1 {
            fmt.Printf("%s: %d\n", word, count)
        }
    }
}
```

统计命令行参数传入文件，所有单词出现次数大于2的。这里使用mutex互斥锁，实现多个goroutine访问map时不会冲突。需要注意的是只有所有的goroutine
 都在等待同一个互斥锁，才能实现这种对同一个资源的竞争。该互斥锁必须是唯一。

## 3.Channel使用

channle可以类比成网络socket，可以单项或者双向的发送数据到接收方。channle发送的数据是有类型的，不同于socket发送的字节流。

```go
package main

import (
    "fmt"
    "os"
    "time"
)

func readStdin(out chan<- []byte) {
    for {
        data := make([]byte, 1024)
        length, _ := os.Stdin.Read(data)
        if length > 0 {
        }
        out <- data
    }
}

func main() {
    done := time.After(30 * time.Second)
    echo := make(chan []byte)
    go readStdin(echo)
    for {
        select {
        case buf := <-echo:
            os.Stdout.Write(buf)
        case <-done:
            fmt.Println("Time out")
            os.Exit(0)
        }
    }

}
```

time.After会在等待的时候后返回一个channel time.Time类型。select会阻塞等到case能执行，多个case都有数据会随机选择。

## 4.如何安全的关闭Channel

```go
package main

import (
    "fmt"
    "time"
)

func send(ch chan string) {
    for {
        ch <- "Hello"
        time.Sleep(500 * time.Millisecond)
    }
}

func main() {
    msg := make(chan string)
    util := time.After(1 * time.Second)
    go send(msg)
    for {
        select {
        case m := <-msg:
            fmt.Println(m)
        case <-util:
            close(msg)
            time.Sleep(500 * time.Microsecond)
            return
        }
    }
}
```

输出:

```go
Hello
Hello
panic: send on closed channel
```

出现panic错误，因为main函数在时间到之后已经关闭了channel，而send后台还要goroutine往msg通道发送数据。注意：关闭通道必须由发送方来关闭否则会出现上面错误。

```go
package main

import (
    "fmt"
    "time"
)

func send(ch chan<- string, done <-chan bool) {
    for {
        select {
        case <-done:
            println("Done")
            close(ch)
            return
        default:
            ch <- "hello"
            time.Sleep(500 * time.Microsecond)
        }
    }
}

func main() {
    msg := make(chan string)
    done := make(chan bool)
    util := time.After(5 * time.Second)
    go send(msg, done)
    for {
        select {
        case m := <-msg:
            fmt.Println(m)
        case <-util:
            done <- true
            time.Sleep(500 * time.Millisecond)
            return
        }
    }
}
```

通过添加一个done通道，来通知send关闭msg通道，实现通道安全关闭。在go当中，经常需要设置一个done通道来进行goroutine之间状态的同步。
上面的用例send函数中done只能用于发送，ch用于接收。接收端触发通道关闭条件时，就需要通知发送端，通过done来通知。

## 5.使用Buffer Cshannel实现锁的功能

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int, lock chan bool) {
    fmt.Printf("%d wants the lock\n", id)
    lock <- true
    fmt.Printf("%d has the lock\n", id)
    time.Sleep(500 * time.Millisecond)
    fmt.Printf("%d is releasing the lock\n", id)
    <-lock
}
func main() {
    lock := make(chan bool, 1)
    for i := 1; i < 7; i++ {
        go worker(i, lock)
    }
    time.Sleep(10 * time.Second)
}
```

输出:

```go
1 wants the lock
1 has the lock
2 wants the lock
4 wants the lock
6 wants the lock
3 wants the lock
5 wants the lock
1 is releasing the lock
2 has the lock
2 is releasing the lock
4 has the lock
4 is releasing the lock
6 has the lock
6 is releasing the lock
3 has the lock
3 is releasing the lock
5 has the lock
5 is releasing the lock
```

在output结果中看到首先是goroutine 1获得写入channel的机会，执行打印操作sleep之后读出channel里的内容，释放channel空间供其他goroutine写入。