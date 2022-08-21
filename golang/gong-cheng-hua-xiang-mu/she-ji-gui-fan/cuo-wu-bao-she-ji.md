# 错误包设计

* 支持错误堆栈
*   能够支持不同的打印格式

    例如%+v、%v、%s等格式，可以根据需要打印不同丰富度的错误信息。
*   能支持 Wrap/Unwrap 功能，也就是在已有的错误上，追加一些新的信息

    Wrap 通常用在调用函数中，调用函数可以基于被调函数报错时的错误 Wrap 一些自己的信息，丰富报错信息，方便后期的错误定位，
*   错误包应该有Is方法

    需要判断某个 error 是否是指定的 error，因为有了 wrapping error，这样判断就会有问题。因为你根本不知道返回的 err 是不是一个嵌套的 error，嵌套了几层
*   ```
    能够支持两种错误创建方式：非格式化创建和格式化创建
    ```

    ```
    errors.New("file not found")
    ```

    errors.Errorf("file %s not found", "iam-apiserver")
*   ```
    错误包应该支持 As 函数
    ```

    ```
    var perr *os.PathError
    if errors.As(err, &perr) {
      fmt.Println(perr.Path)
    }
    ```







