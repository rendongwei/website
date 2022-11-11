---
title: "Go-Json"
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

很多程序都需要处理或者发布数据，不管这个程序是要使用数据库，进行网络调用，还是与分布式系统打交道。如果程序需要处理XML或者JSON，可以使用标准库名为xml和json的包，它们可以处理这些格式的数据。如果想实现自己的编解码，可以将这些包的实现作为指导。

在今天，JSON远比XML流行。这主要是因为与XML相比，使用JSON需要处理的标签更少。而这意味着网络传输时每个消息的数据更少传输效率更高，提升系统性能。而且JSON可以转换为BSON(Binary JavaScript Object Notation，二进制JavaScript对象标记)，进一步缩小每个消息的数据长度。因此，我们学习如何在Go程序里处理并发布JSON。处理XML的方法也很类似。

## 1.解码JSON

```go
package main

import (
    "encoding/json"
    "fmt"
    "io"
    "log"
    "strings"
)

type User struct {
    UserName string `json:"username"`
    Password string `json:"password"`
}

var jsonString string = `{
    "username":"aaa@163.com",
    "password":"12356"
}`

func main() {
    var user User
    reader := strings.NewReader(jsonString)
    err := json.NewDecoder(reader).Decode(&user) 
    if err != nil {
        log.Fatalln(err)
    }
    fmt.Printf("%#v", user)
}
```

输出:

```go
main.User{UserName:"aaa@163.com", Password:"12356"}
```

NewDecoder函数接收一个实现了io.Reader接口类型的值作为参数。只要类型实现了这些接口，就可以自动获得许多功能支持。函数NewDecoder返回一个指向Decoder类型的指针。由于Go语言支持复合语句调用，可以直接调用从NewDecoder函数返回值的Decode方法，而不用将这个返回值存入变量。func (dec *Decoder) Decode(v interface{}) error  而Decode方法接收一个interface{}类型的值做参数，并返回一个error值。因为任何类型都实现了一个空接口interface{}。这意味着Decode方法可以接收任何类型的值。使用反射，Decode方法会拿到传入值的类型信息。然后，在读取JSON数据流的时候，Decode方法会解码成对应的类型值。我们向Decode方法传入的指向user类型的指针变量地址，而这个地址实际值为nil。该方法调用后，这个指针变量被赋给一个User类型的值，并根据解码后的JSON文档做初始化。

有时，需要处理的JSON文档会以String的形式存在。在这种情况下，需要将string转为切片[]byte，并使用json包的Unmarshal函数进行反序列化处理。如下代码：

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
)

type Contact struct {
    Name    string `json:"name"`
    Title   string `json:"title"`
    Contact struct {
        Home string `json:"home"`
        Cell string `json:"cell"`
    } `json:"contact"`
}

//JSON包含用于反序列号的演示字符串
var JSON = `{
"name": "Gopher",
"title":"programer",
"contact":{"home":"415.333.33333", "cell":"415.55.555"}
}`

func main() {
    var c Contact
    err := json.Unmarshal([]byte(JSON), &c)
    if err != nil {
        log.Fatal("Error:", err)
        return
    }
    fmt.Println(c)
}
```

以上例子将JSON文档保存在一个字符串变量里，并使用Unmarshal函数将JSON文档解码到一个结构类型的值里。

有时，无法为JSON的格式声明一个结构类型，而是需要更加灵活的方式来处理JSON文档，在这种情况下，可以将JSON文档解码到一个map变量中：

```go
var JSON = `{
"name": "Gopher",
"title":"programer",
"contact":{"home":"415.333.33333", "cell":"415.55.555"}
}`

func main() {
    var c map[string]interface{}
    err := json.Unmarshal([]byte(JSON), &c)
    if err != nil {
        log.Fatal("Error:", err)
        return
    }
    fmt.Println(c)
    fmt.Println("Name:", c["name"])
    fmt.Print(c["contact"].(map[string]interface{})["cell"])
}
```

输出:

```go
map[contact:map[cell:415.55.555 home:415.333.33333] name:Gopher title:programer]
Name: Gopher
415.55.555
```

将结构类型变量替换为map类型的变量。变量c声明为一个map类型，键是string类型，其值是interface{}类型。这意味着map类型可以使用任意类型的值作为给定键的值。虽然这种方法为处理JSON文档带来很大灵活性，但是有一个缺点。让我们看一下访问contact子文档的cell字段代码：

```go
fmt.Print(c["contact"].(map[string]interface{})["cell"])
```

因为每个键的值类型都是interface{}，所以必须将值转为合适的类型，才能处理这个值。

## 2.编码JSON

以上举例说明了将json文档解码为json类型，下面看下如何将json类型变量转为json文档。使用json包MarshalIndent函数可以进行编码。该函数可以很方便地将Go语言map类型的值或者结构类型转换为易读格式的json文档。序列号是指将数据转换为json字符串的过程。

```go
package main

import (
    "encoding/json"
    "fmt"
    "log"
)

func main() {
    c := make(map[string]interface{})
    c["name"] = "Gopher"
    c["title"] = "programmer"
    c["contact"] = map[string]interface{}{
        "home": "415.55.555",
        "cell": "413.33.333",
    }
    //将这个映射序列化到Json字符串
    data, err := json.MarshalIndent(c, "", " ")
    if err != nil {
        log.Fatal("Error:", err)
        return
    }
    fmt.Println(string(data))
}
```

输出:

```go
{
 "contact": {
  "cell": "413.33.333",
  "home": "415.55.555"
 },
 "name": "Gopher",
 "title": "programmer"
}
```

以上代码展示了json包的MarshalIndent函数将一个map值转换为JSON字符串。函数MarshalIndent返回一个byte切片，用来保存JSON字符串和一个error值。

下面看下json包MarshalIndent函数：

```go
func MarshalIndent(v interface{}, prefix, indent string) ([]byte, error) {
    b, err := Marshal(v)
    if err != nil {
        return nil, err
    }
    var buf bytes.Buffer
    err = Indent(&buf, b, prefix, indent)
    if err != nil {
        return nil, err
    }
    return buf.Bytes(), nil
}
```

MarshalIndent解码最终调用的还是Marshal，只是用缩进对输出进行格式化了。Marshal会使用反射来确定如何将map类型转换为JSON字符串。如果不需要输出带缩进格式JSON字符串，直接使用Marshal即可。