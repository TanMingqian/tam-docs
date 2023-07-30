# Makefile

Makefile 的规则语法，主要包括 target、prerequisites 和 command

target ...: prerequisites ... command ... ...&#x20;

• target，可以是一个 object file（目标文件），也可以是一个执行文件，还可以是一个标签（label）。target 可使用通配符，当有多个目标时，目标之间用空格分隔。&#x20;

• prerequisites，代表生成该 target 所需要的依赖项。当有多个依赖项时，依赖项之间用空格分隔。 command，代表该 target 要执行的命令（可以是任意的 shell 命令）。



### 伪目标

```makefile
.PHONY: clean
clean:
    rm hello.o
```

### 命令

Makefile 支持 Linux 命令，调用方式跟在 Linux 系统下调用命令的方式基本一致。默认情况下，make 会把正在执行的命令输出到当前屏幕上。但我们可以通过在命令前加@符号的方式，禁止 make 输出当前正在执行的命令。

```makefile
.PHONY: test
test:
    @echo "hello world"

```

### 变量

```
= 最基本的赋值方法
BASE_IMAGE = alpine:3.10
A = a
B = $(A) b
A = c
B 最后的值为 c b，而不是 a b。也就是说，在用变量给变量赋值时，右边变量的取值，取的是最终的变量值。


:=直接赋值，赋予当前位置的值
A = a
B := $(A) b
A = c
B 最后的值为 a b。通过 := 的赋值方式，可以避免 = 赋值带来的潜在的不一致


?= 表示如果该变量没有被赋值，则赋予等号后的值
PLATFORMS ?= linux_amd64 linux_arm64

+=表示将等号后面的值添加到前面的变量上
MAKEFLAGS += --no-print-directory

```

### 环境变量

有两种环境变量，分别是 Makefile 预定义的环境变量和自定义的环境变量 ... export USAGE\_OPTIONS … 默认情况下，Makefile 中定义的环境变量只在当前 Makefile 有效，如果想向下层传递（Makefile 中调用另一个 Makefile），需要使用 export 关键字来声明。

### 特殊变量

![](<../../../.gitbook/assets/image (6) (2).png>)

### 自动化变量

![](<../../../.gitbook/assets/image (35).png>)

### 条件语句

```
# if ...
<conditional-directive>
<text-if-true>
endif
# if ... else ...
<conditional-directive>
<text-if-true>
else
<text-if-false>
endif

• ifeq：条件判断，判断是否相等

• ifneq：条件判断，判断是否不相等

• ifdef：条件判断，判断变量是否已定义

• ifndef：条件判断，判断变量是否未定义
```

### 函数

```makefile
define 函数名
函数体
endef
```

### 预定义函数

![](<../../../.gitbook/assets/image (32).png>)

### 引入其他 Makefile

在项目根目录下的 Makefile 中引入其他 Makefile

```makefile
include scripts/make-rules/common.mk
include scripts/make-rules/golang.mk

```

### 《跟我一起写 Makefile》 (PDF 重制版)

{% embed url="https://github.com/seisman/how-to-write-makefile" %}

