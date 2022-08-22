# defer

defer 关键字后面的表达式，是在将 deferred 函数注册到 deferred 函数栈的时候进行求值的​

```go

func foo1() {
    for i := 0; i <= 3; i++ {
        defer fmt.Println(i)
    }
}

func foo2() {
    for i := 0; i <= 3; i++ {
        defer func(n int) {
            fmt.Println(n)
        }(i)
    }
}

func foo3() {
    for i := 0; i <= 3; i++ {
        defer func() {
            fmt.Println(i)
        }()
    }
}

func main() {
    fmt.Println("foo1 result:")
    foo1()
    fmt.Println("\nfoo2 result:")
    foo2()
    fmt.Println("\nfoo3 result:")
    foo3()
}
```





