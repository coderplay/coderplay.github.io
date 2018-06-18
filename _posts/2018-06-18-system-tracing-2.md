---
layout: post
title: 系统动态追踪(二)
key: 20180618
tags:
  - Dynamic Tracing
---

上一篇我们看到动态追踪的威力，但这里存在一个矛盾点：大部分时候这种动态追踪需要用到debuginfo, 而在生产环境基本没有哪个机器会安装debuginfo。并且debuginfo通常很大， 因为它包含了所有函数的源码、行号、参数等信息。有的商业程序甚至不提供debuginfo，例如Oracle的Hotspot JVM。上面几个原因使得很多时候在生产环境拥有debuginfo不大现实。 那么没有debuginfo的情况下，我们怎么在生产系统进行动态追踪呢？这是此篇需要解决的问题。

## 基础知识
要回答生产系统如何进行动态追踪，需要简单地了解以下的一些基础知识。由于动态追踪关心的一般是函数相关的探点，所以这里重点介绍函数相关的知识。下面看一个简单的Hello world程序`hello.asm`。

```assembly
;;; 编译方法
;;; nasm -f elf64 hello.asm
;;; ld -o hello hello.o
section .data
        hello_world db "Hello, world!", 10
        hello_dt db "Hello, Dynamic Tracing!", 10

section .text
        global _start

_start:
        mov r8, hello_world ; 将hello_world串放入r8寄存器
        mov r9, 14          ; 将hello_world串的长度14放入r9寄存器
        call print_hello    ; 第一次调用print_hello

        mov r8, hello_dt    ; 将hello_dt串放入r8寄存器
        mov r9, 24          ; 将hello_dt串的长度24放入r9寄存器
        call print_hello    ; 第二次调用print_hello

        mov rax, 60         ; sys_exit
        mov rdi, 0          ; error_code = 0
        syscall             ; 调用sys_exit(0)退出程序

print_hello:
        mov rax, 1          ; sys_write
        mov rdi, 1          ; fd = STDOUT
        mov rsi, r8         ; 将r8的内容放入rsi寄存器
        mov rdx, r9         ; 将r9的内容放入rdx寄存器
        syscall             ; 调用sys_write(STDOUT, 串, 串长度)打印串
        ret                 ; 函数返回
```

此程序采用`Intel`汇编语法编写。程序开始定义了两个串 `Hello, world!`和`Hello, Dynamic Tracing!`，接着定义了程序的起始函数`_start`。汇编程序总是以`_start`为程序起始函数名，就像c和java程序总是以main为起始函数名一样。`_start`函数通过`call`指令调用了两次`print_hello`函数将上面定义的两串分别在标准输出打印出来，最后退出。

`print_hello`的输入有两个参数：第一个参数是要打印的串，第二个参数是串的长度。

那么`_start`函数怎么将串和串的长度传给`print_hello`函数呢？它可以通过寄存器，也可以将两参数压栈。当然寄存器更高效。x86有8个通用寄存器 : `eax`, `ebx`, `ecx`, `edx`, `ebp`, `esp`, `esi`, `edi`。x64把它们扩展到64位，分别用`r`前缀代替`e`前缀，同时另外再添加了8个通用寄存器：`r8`, `r9`, `r10`, `r11`, `r12`, `r13`, `r14`, `r15`。那么问题来了， 这么多寄存器该用哪两个向`print_hello`传递参数？此处我们随机使用了`r8`, `r9`。`r8`寄存了串的地址。`r9`寄存了串的长度，是一个立即数。当然你也可以使用`rax`，`rbx`或`r14`, `r15`。这16个寄存器随便你选两个，只是要注意不要覆盖掉`syscall`指令需要的寄存器上的值就行。

### 调用约定 (Calling Convention)
接着另一个问题就来了：如果函数间使用随机的寄存器交换数据， 事情就会变得异常复杂。 编程人员需要记住每个被调函数用哪个寄存器接收第几个参数，用哪个寄存器存储主调函数的地址。这对于动态链接的共享库来说就变得更不可接受。共享库的作者和调用库的程序作者可以是不同的人在不同的时间编写的，甚至可以是采用不同的编程语言编写的。基于这种原因，编写软件时需要遵行一种函数调用的规范：规定主调函数和被调函数之间怎么分配寄存器保留现场、返回现场、传递参数、返回值等操作。Linux 86_64上使用了`System V AMD64`的调用约定统一这些规则。例如：第一个参数放在`rdi`寄存器，第二个在`rsi`， 第三个在`rdx`， 然后是`rcx`，`r8`和`r9`。第六个之后的参数则放在栈上。函数的返回值则在`rax`寄存器中。下面看一个例子：

```c
#include <stdio.h>

int sum(int a ,int b, int c, int d,
        int e, int f, int g, int h)
{
    return a + b + c + d + e + f + g + h;
}

int main()
{
    int s = sum(2, 0, 1, 8, 0, 5, 2, 3);
    printf("%d", s);
}
```
这里有一个在线编译的[网站](https://godbolt.org/g/EinBGr)，大家可以尝试修改上面c代码看看修改后生成的汇编代码。或者本地通过`gcc -o sum sum.c`编译，用`objdump -d sum`查看生成的汇编。下面截取其中一段`call`指令之前的准备汇编，我们通常称之为函数序幕(function prologue)

```assembly
pushq $3
pushq $2
movl  $5,  %r9d
movl  $0,  %r8d
movl  $8,  %ecx
movl  $1,  %edx
movl  $0,  %esi
movl  $2,  %edi
call  sum
```
 小结一下：寄存器约定可见下表：


| 寄存器  | 用途  |
|----|-------|
|rax|函数返回值|
|rcx|第4个参数|
|rdx|第3个参数|
|rsi|第2个参数|
|rdi|第1个参数|
|rbp|栈底|
|rsp|栈顶|
|r8|第5个参数|
|r9|第6个参数|
|rbx, r10-r15|任意|



`rbp`  is the frame pointer on x86_64. In your generated code, it gets a snapshot of the stack pointer (`rsp`) so that when adjustments are made to  `rsp`  (i.e. reserving space for local variables or  `push`ing values on to the stack), local variables and function parameters are still accessible from a constant offset from  `rbp`.

A lot of compilers offer frame pointer omission as an optimization option; this will make the generated assembly code access variables relative to  `rsp`  instead and free up  `rbp`  as another general purpose register for use in functions.

In the case of GCC, which I'm guessing you're using from the AT&T assembler syntax, that switch is  `-fomit-frame-pointer`. Try compiling your code with that switch and see what assembly code you get. You will probably notice that when accessing values relative to  `rsp`  instead of  `rbp`, the offset from the pointer varies throughout the function.



### 符号(Symbol)
机器的语言是二进制的，它可以完全不需要符号。例如函数名、变量名和类型名称， 对于机器来说只需要知道其地址就够了。符号是用来给人类理解程序的。如果程序没有符号，光看二进制的地址基本无法理解程序的结构。并且符号对应的地址是在程序链接时候决定的， 如果程序使用动态链接技术，没有符号就更麻烦。

### ELF文件格式
ELF ，全称Executable and Linkable Format，是Linux操作系统上用在可执行文件、目标文件、core dump、标准的文件格式。我们在Linux上

### dwarf

### 进程内存布局(Process Memory Layout)


## k[ret]probe及u[ret]probe
接着谈谈动态追踪技术的实现。`perf`是一个工具集，动态追踪部分是由


## 无debuginfo的生产环境动态追踪

### 追踪函数参数
### 追踪函数返回值 
### 追踪复杂结构的参数
### 巧用perf估算追踪地址
### c++程序
下面来看看c++程序的追踪。 c++支持函数重载，即多个成员函数可以拥有相同的名称但接收不同的参数。那么c++编译器在生成对象文件时用什么机制区分这些函数呢？
#### 成员函数追踪
mangling
#### 成员变量追踪
### go程序

