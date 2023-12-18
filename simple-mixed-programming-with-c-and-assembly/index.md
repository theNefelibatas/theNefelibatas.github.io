# 简单的 C 语言与汇编语言混合编程


<!--more-->

## 任务

给定函数：

$$
\begin{aligned}
 F(n) &= 
 \begin{cases}
  n & \textrm{if } n=0, 1, 2 \\\\
  F(n-1)+F(n-2) & \textrm{if } n \geq 3 \textrm{, } n\in\mathbb{N}
 \end{cases}
\end{aligned}
$$

用汇编语言编写计算上述数列的子程序 `ditui1`，用C语言函数来获取输入、调用 `ditui1` 汇编函数，并且显示子程序返回的结果。

## 环境

编程环境：WSL2 Ubuntu22.04 64 位

编辑器：VSCode

编译器信息如下：

```bash
$ gcc --version
gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

$ nasm --version
NASM version 2.15.05
```

## 解决方案

编写 `main.c` 处理输入并进行输出，编写 `fibonacci.asm` 完成子程序 `ditui1`

```
simple-mixed-programming
│
├── fibonacci.asm
│
└── main.c
```

编译命令如下，基于 32 位平台：

```bash
$ nasm -f elf32 fibonacci.asm -o fibonacci.o
$ gcc -m32 main.c fibonacci.o -o fibonacci
```

`main.c` 代码如下：

```c title="main.c"
#include <stdio.h>

// 声明外部的汇编函数
extern int ditui1();

int main() {
    int n, result;

    // 输入
    printf("Enter a number: ");
    scanf("%d", &n);

    // 调用汇编函数
    result = ditui1(n);

    // 输出
    printf("F(%d) = %d\n", n, result);

    return 0;

}

// 0, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, ...
```

在 x86 架构的汇编语言中，如 8086、80286、80386 等，寄存器是 16 位的，可以通过拆分成 8 位的高低字节进行操作。例如，`ax` 寄存器可以拆分为 `ah`（高字节）和 `al`（低字节），`bx` 寄存器可以拆分为 `bh` 和 `bl`，以此类推。

在 32 位和 64 位的 x86 架构中，如 Intel Pentium、Core i7 和 AMD64 等处理器中，寄存器是 32 位的（对于 64 位架构，寄存器是 64 位的）。在这些架构中，`eax` 是一个 32 位的寄存器，不支持直接拆分成 `eal` 和 `eah`。

因此在 `fibonacci.asm` 中将直接使用 32 位寄存器，代码如下：

```assembly title="fibonacci.asm"
section .text
global ditui1

ditui1:

    push ebx
    push ecx
    push edx

    ; 检查输入n是否为0或1
    cmp eax, 0  
    je end
    cmp eax, 1
    je end

    ; 初始化
    mov ecx, eax
    xor eax, eax
    mov ebx, 0 ;; F(0)
    mov edx, 1 ;; F(1)

iteration:
    mov eax, edx   ; 将前一个数F(n-1)保存到eax寄存器

    add edx, ebx   ; 计算下一个斐波那契数

    mov ebx, eax   ; 更新F(n-2)

    loop iteration  ; 继续循环

    mov eax, edx

end:

    pop edx
    pop ecx
    pop ebx
    ret            ; 返回
```

直接执行上面的命令会出现如下错误：

```bash
$ nasm -f elf32 fibonacci.asm -o fibonacci.o
$ gcc -m32 main.c fibonacci.o -o fibonacci
In file included from main.c:1:
/usr/include/stdio.h:27:10: fatal error: bits/libc-header-start.h: No such file or directory
   27 | #include <bits/libc-header-start.h>
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
```

参考 StackOverflow 上的这个提问：[c - "fatal error: bits/libc-header-start.h: No such file or directory" while compiling HTK - Stack Overflow](https://stackoverflow.com/questions/54082459/fatal-error-bits-libc-header-start-h-no-such-file-or-directory-while-compili)

在 64 位 Ubuntu 上 `gcc` 通常只有 64 位工具，因此需要安装 32 位工具，执行以下命令：

```bash
$ sudo apt-get install gcc-multilib
```

再编译运行即可：

```bash
$ nasm -f elf32 fibonacci.asm -o fibonacci.o
$ gcc -m32 main.c fibonacci.o -o fibonacci
$ ./fibonacci
```

