# go程序启动

Go 程序由一系列 Go 包组成，代码的执行也是在各个包之间跳转。和其他语言一样，Go 也拥有自己的用户层入口：main 函数

### main.main 函数：Go 应用的入口函数

```go
package main
​
func main() {
    // 用户层执行逻辑
    ... ...
}
```

Go 语言要求：**可执行程序的 main 包必须定义 main 函数**，否则 Go 编译器会报错。

在启动了多个 Goroutine的 Go 应用中，main.main 函数将在 Go 应用的主 Goroutine 中执行

对于 main 包的 main 函数来说，虽然是用户层逻辑的入口函数，但它却**不一定是用户层第一个被执行的函数**。

### init 函数：Go 包的初始化函数

```go
func init() {
    // 包初始化逻辑
    ... ...
}
```

main 包依赖的包中定义了 init 函数，或者是 main 包自身定义了 init 函数，那么 Go 程序在这个包初始化的时候，就会自动调用它的 init 函数

Go 包可以拥有不止一个 init 函数，每个组成 Go 包的 Go 源文件中，也可以定义多个 init 函数。

在初始化 Go 包时，Go 会按照一定的次序，逐一、顺序地调用这个包的 init 函数。一般来说，先传递给 Go 编译器的源文件中的 init 函数，会先被执行；而同一个源文件中的多个 init 函数，会按声明顺序依次执行

### Go 包的初始化次序

![](<../../../.gitbook/assets/image (3).png>)

**依赖包按“深度优先”的次序进行初始化；**

**每个包内按以“常量 -> 变量 -> init 函数”的顺序进行初始化；**

**包内的多个 init 函数按出现次序进行自动调用。**






