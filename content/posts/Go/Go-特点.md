---
title: "Go-特点"
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

# 支持函数多个返回值

```go
package main

import "fmt"

//定义两个返回值的函数
func getNames() (n1, n2 string) {  
    return "name1", "name2"
}

func main() {
    n1, n2 := getNames()
    fmt.Println(n1, n2)
    
    //选择一个返回值，忽略另一个
    n3, _ := getNames()  
    fmt.Println(n3)
}
```

也可以直接对形参赋值，return后面直接返回即可

```go
func getNames() (n1, n2 string) {
    n1 = "name1"
    n2 = "name2"
    return
}
```

# go包含大量的标准库

```go
package main

import (
    "bufio"
    "fmt"
    "net"
)

func main() {
    //通过TCP建立连接
    conn, _ := net.Dial("tcp", "www.baidu.com:80")
    //向网络连接发送数据
    fmt.Fprintf(conn, "GET / HTTP/1.0\r\n\r\n")
    //打印响应的第一行
    status, _ := bufio.NewReader(conn).ReadString('\n')  
    fmt.Println(status)
}
```

go发起网络连接很简单，同时启动一个web服务器同样也简单。借助标准库，如下所示：

```go
package main

import (
    "fmt"
    "io/ioutil"
    "net/http"
)

func main() {
    res, _ := http.Get("http://www.baidu.com")
    body, _ := ioutil.ReadAll(res.Body)
    fmt.Println(string(body))
    res.Body.Close()
}
```

以上显示了http请求用例。

# Testing

在go语言里面，使用testing包可以很容易的写测试用例，这在软件开发当中非常重要：

```go
func getName() (string, string) {
    return "name1", "name2"
}

func TestNames(t *testing.T) {
    s, s2 := getName()
    if s != "name1" || s2 != "name2" {
        t.Error("error occurred from name")
    }
}
```

测试用例go文件命名以_test结尾，测试函数名以Test开头。在命令行执行go test即可执行测试用例。

