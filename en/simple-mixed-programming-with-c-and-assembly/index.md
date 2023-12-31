# Simple Mixed Programming With C and Assembly Languages


<!--more-->

## Task

Given the function:

$$
\begin{aligned}
 F(n) &= 
 \begin{cases}
  n & \textrm{if } n=0, 1, 2 \\\\
  F(n-1)+F(n-2) & \textrm{if } n \geq 3 \textrm{, } n\in\mathbb{N}
 \end{cases}
\end{aligned}
$$

Write an assembly subroutine `ditui1` to compute the above sequence. Use C functions to obtain the input, call the assembly function `ditui1`, and display the results returned by the subroutine.

## Environment

Programming Environment: WSL2 Ubuntu22.04 64-bit

Editor: VSCode

Compiler versions are as follows:

```bash
$ gcc --version
gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

$ nasm --version
NASM version 2.15.05
```

## Solution

Write `main.c` to handle input and output and create `fibonacci.asm` to define the subroutine `ditui1`.

```
simple-mixed-programming
│
├── fibonacci.asm
│
└── main.c
```

To compile based on 32-bit platform:

```bash
$ nasm -f elf32 fibonacci.asm -o fibonacci.o
$ gcc -m32 main.c fibonacci.o -o fibonacci
```

The `main.c` code is:

```c title="main.c"
#include <stdio.h>

// Declaration of the external assembly function
extern int ditui1();

int main() {
    int n, result;

    // Get the input
    printf("Enter a number: ");
    scanf("%d", &n);

    // Call the assembly function
    result = ditui1(n);

    // Display the result
    printf("F(%d) = %d\n", n, result);

    return 0;
}

// 0, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, ...
```

In x86 assembly for architectures such as 8086, 80286, and 80386, registers are 16-bit and can be split into 8-bit high and low bytes for operations. For instance, the `ax` register can be divided into `ah` (high byte) and `al` (low byte), and so on.

In 32-bit and 64-bit x86 architectures, like Intel Pentium, Core i7, and AMD64 processors, registers are 32-bit (64-bit for 64-bit architecture). In these architectures, `eax` is a 32-bit register, and there isn't a direct split into `eal` and `eah`.

Thus, the `fibonacci.asm` will use 32-bit registers directly. The code is:

```assembly title="fibonacci.asm"
section .text
global ditui1

ditui1:

    push ebx
    push ecx
    push edx

    ; Check if the input n is 0 or 1
    cmp eax, 0  
    je end
    cmp eax, 1
    je end

    ; Initialization
    mov ecx, eax
    xor eax, eax
    mov ebx, 0 ;; F(0)
    mov edx, 1 ;; F(1)

iteration:
    mov eax, edx   ; Save the previous number F(n-1) to the eax register

    add edx, ebx   ; Compute the next Fibonacci number

    mov ebx, eax   ; Update F(n-2)

    loop iteration  ; Continue the loop

    mov eax, edx

end:

    pop edx
    pop ecx
    pop ebx
    ret            ; Return
```

Executing the above commands would yield the following error:

```bash
$ nasm -f elf32 fibonacci.asm -o fibonacci.o
$ gcc -m32 main.c fibonacci.o -o fibonacci
In file included from main.c:1:
/usr/include/stdio.h:27:10: fatal error: bits/libc-header-start.h: No such file or directory
   27 | #include <bits/libc-header-start.h>
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
```

Refer to this question on StackOverflow: [c - "fatal error: bits/libc-header-start.h: No such file or directory" while compiling HTK - Stack Overflow](https://stackoverflow.com/questions/54082459/fatal-error-bits-libc-header-start-h-no-such-file-or-directory-while-compili)

On 64-bit Ubuntu, gcc normally only comes with 64-bit libraries. So you need to install 32-bit headers and libraries. Try to execute the following command:

```bash
$ sudo apt-get install gcc-multilib
```

Then compile and run:

```bash
$ nasm -f elf32 fibonacci.asm -o fibonacci.o
$ gcc -m32 main.c fibonacci.o -o fibonacci
$ ./fibonacci
```

