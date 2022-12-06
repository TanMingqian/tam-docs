# defer

**defer 是 Go 语言提供的一种延迟调用机制**

defer 关键字后面只能接函数（或方法），这些函数被称为 deferred 函数。

defer 将它们注册到其所在 Goroutine 中，用于存放 deferred 函数的栈数据结构中，这些 deferred 函数将在执行 defer 的函数退出前，按后进先出（LIFO）的顺序被程序调度执行

![](<../../../.gitbook/assets/image (4) (3).png>)

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

foo1 中 defer 后面直接用的是 fmt.Println 函数，每当 defer 将 fmt.Println 注册到 deferred 函数栈的时候，都会对 Println 后面的参数进行求值。根据上述代码逻辑，依次压入 deferred 函数栈的函数是：

```go
fmt.Println(0)
fmt.Println(1)
fmt.Println(2)
fmt.Println(3)
```

因此，当 foo1 返回后，deferred 函数被调度执行时，上述压入栈的 deferred 函数将以 LIFO 次序出栈执行，这时的输出的结果为：

```go
3
2
1
0
```

foo2 中 defer 后面接的是一个带有一个参数的匿名函数。每当 defer 将匿名函数注册到 deferred 函数栈的时候，都会对该匿名函数的参数进行求值。根据上述代码逻辑，依次压入 deferred 函数栈的函数是：

```go
func(0)
func(1)
func(2)
func(3)


3
2
1
0
```

foo3 中 defer 后面接的是一个不带参数的匿名函数。根据上述代码逻辑，依次压入 deferred 函数栈的函数是：

```go
func()
func()
func()
func()
```

当 foo3 返回后，deferred 函数被调度执行时，上述压入栈的 deferred 函数将以 LIFO 次序出栈执行。匿名函数会以闭包的方式访问外围函数的变量 i，并通过 Println 输出 i 的值，此时 i 的值为 4，因此 foo3 的输出结果为：

```go
4
4
4
4
```
