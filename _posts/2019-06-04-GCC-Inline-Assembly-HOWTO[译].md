---
layout: post
title: "GCC Inline Assembly HOWTO[译]"
comments: true
description: "GCC Inline Assembly HOWTO[译]"
keywords: "inline assembly, c"
---

由于最近在完成 JOS 的 lab3，需要用到不少的内联汇编的知识，准备集中学习一下，将其中一篇讲得比较好的文章翻译下来，在译文的基础上删掉一些啰嗦的地方，也添加了一些自己的理解。

原文链接：[GCC-Inline-Assembly-HOWTO](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)

&nbsp;

### 一、介绍

___

略。

&nbsp;

### 二、预览

___

我们在在这里学习内联汇编。那内联汇编到底是什么？

我们可以指导编译器，在一个函数被调用时，让它整个函数体插进它的 caller 的函数内，这种函数称为[内联函数](https://en.wikipedia.org/wiki/Inline_function)。听起来和 C 语言中的[Macro](https://en.wikipedia.org/wiki/Macro_(computer_science))类似。

那什么是内联汇编呢？内联汇编就是函数里包含一些汇编例程，在系统编程时能加快程序的运行。我们主要关注 GCC 内联汇编以及它的格式与用法。我们使用关键字 **asm** 来声明内联汇编。

内联汇编之所以重要，是因为它的输出能通过 C 语言变量表现出来。因为具备这种能力，内联汇编成为了汇编函数与 C 程序之间的桥梁。

&nbsp;

### 三、汇编语法

___

[GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)，是 [GUN](https://en.wikipedia.org/wiki/GNU) 提供的 linux 中 C 语言的编译器，使用了 **AT&T/UNIX** 两种汇编语法。我们主要以 AT&T 格式来进行讲解。下面介绍一下二者语法上的区别。

#### 3.1、源-目的 顺序

AT&T 与 UNIX 的源/目的寄存器位置恰好相反：

Intel: Op-code dst, src

AT&T: Op-code src, dst

#### 3.2、寄存器名

Intel 语法中使用的 eax 寄存器，拿到 AT&T 就变成了以 **%** 为前缀的 %eax。

#### 3.3、立即数

在 AT&T 中立即数以 **$** 为前缀，十六进制常数以 **0x** 为前缀；而 Intel 语法则没有前缀，十六进制数以 **h** 为后缀。

#### 3.4、操作数大小

在 AT&T 语法中，操作数大小以操作码的最后一个字母标识：**'b'**(8-bit), **'w'**(16-bit), **'l'**(32-bit)。如果使用 Intel 语法，则在内存操作数前以 **'byte ptr'**, **'word ptr'**, **'dword ptr'** 标识。

Intel: mov al, byte ptr foo
AT&T: movb foo, %al

#### 3.5、内存操作数

例子：

```
+------------------------------+------------------------------------+
|       Intel Code             |      AT&T Code                     |
+------------------------------+------------------------------------+
| mov     eax,1                |  movl    $1,%eax                   |   
| mov     ebx,0ffh             |  movl    $0xff,%ebx                |   
| int     80h                  |  int     $0x80                     |   
| mov     ebx, eax             |  movl    %eax, %ebx                |
| mov     eax,[ecx]            |  movl    (%ecx),%eax               |
| mov     eax,[ebx+3]          |  movl    3(%ebx),%eax              | 
| mov     eax,[ebx+20h]        |  movl    0x20(%ebx),%eax           |
| add     eax,[ebx+ecx*2h]     |  addl    (%ebx,%ecx,0x2),%eax      |
| lea     eax,[ebx+ecx]        |  leal    (%ebx,%ecx),%eax          |
| sub     eax,[ebx+ecx*4h-20h] |  subl    -0x20(%ebx,%ecx,0x4),%eax |
+------------------------------+------------------------------------+
```

注意上面 AT&T 格式中，在引用内存时，常量没有使用 **$** 前缀。

&nbsp;

### 四、基本内联汇编

___

基本内联的格式还是挺简单：

```
asm("assembly code");
```

例1：

```
asm("movl %ecx %eax"); 
__asm__("movb %bh (%eax)");
```

注意使用 **asm** 与 **_asm_** 是等价的，当 asm 在我们的程序中有同名冲突时，就可以使用 _asm_。

也可以在同一个 asm 里写多个指令，例如：

```
__asm__("movl %eax, %ebx\n\t"
        "movl $56, %esi\n\t"
        "movl %ecx, $label(%edx,%ebx,$4)\n\t"
        "movb %ah, (%ebx)");
```

如果我们通过内联汇编改变了某些寄存器的值，在内联汇编结束时却没有去恢复这些值，就会出现不好的情况。因为 GCC 无从知道哪些寄存器的内容发生了改变，尤其是编译器想要做一些优化的时候，更有可能出问题。在写内联汇编的时候，通常会将某个寄存器与 C 语言中的某个变量结合起来。对变量值的改变就会造成对相应寄存器值的改变。如果我们不这些变化通知给 GCC 的话，GCC 将会无视这些值的变化，导致的结果就是程序的执行完全偏离我们的预想。

在这种情况下，我们能做的就是不去使用那些带有副作用的指令，或者在结束时恢复寄存器原来的值。为了解决这些问题，提高内联汇编的作用，推出了一种新的形式：扩展内联汇编。

&nbsp;

### 五、扩展内联汇编

___

在扩展内联里，我们可以指定操作数。例如指定输入寄存器（Input Register），输出寄存器（Output Register）和破坏寄存器（Clobbered Register）。GCC 允许编程人员不指定特定的寄存器，而是自己（GCC）通过优化的算法来选择合适的寄存器。下面是扩展内联汇编的格式：

```
asm (assembler template 
    :output operands                  /* optional */
    :input operands                   /* optional */
    :list of clobbered registers      /* optional */
    );
```

**assembler template** 是一些汇编指令，一条或者多条。**output operands** 是输入操作数，多个操作数以逗号隔开。**input operands** 输出操作数同理。操作数的个数通常与 **assembler template** 中的指令有关。

例2：

```
asm ("cld\n\t"
     "rep\n\t"
     "stosl"
     : /* no output registers */
     : "c" (count), "a" (fill_value), "D" (dest)
     : "%ecx", "%edi" 
     );
```

这条内联汇编，将 fill_value 的 count 倍的值存进 DS:EDI（edi的值由dest指定） 所指的地方。它还告诉 GCC，ecx 与 edi 寄存器的内容发生了改变。

例3：

```

```