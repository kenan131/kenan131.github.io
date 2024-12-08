---
title: GO泛型
date: 2024-09-22 17:11:04
tags: GO
categories: GO
---

**Go 1.18 或更高版本的安装.** 有关安装说明，请参阅 [安装 Go](https://go.p2hp.com/doc/install).

```go
// Object 泛型父类
type Object interface {
    *One | *Two
    TestMethod() int // 子类必须实现该接口，否则无法识别
}

type One struct {
    Age  int
    Name string
}

type Two struct {
    Age int
}

func (o *One) TestMethod() int {
    return o.Age
}

func (t *Two) TestMethod() int {
    return t.Age
}

func test[V Object](v V) int {
    res := v.TestMethod()
    return res
}

var sm sync.WaitGroup

func main() {
    one := &One{Age: 18}
    v := test(one)
    fmt.Println(v)

    two := &Two{Age: 28}
    v = test(two)
    fmt.Println(v)
}
```

参考文档

https://go.p2hp.com/go.dev/doc/tutorial/generics