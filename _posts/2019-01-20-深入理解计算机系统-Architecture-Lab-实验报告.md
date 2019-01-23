---
layout: post
title: "深入理解计算机系统 Architecture Lab 实验报告"
comments: true
description: "深入理解计算机系统 Architecture Lab 实验报告"
keywords: "Architecture Lab, 深入理解计算机系统, CSAPP"
---

## 1 介绍

通过这个实验，将会学习到流水线化的 Y86-64 处理器的设计和实现。通过优化，使程序的性能最大化。当你完成这个实验后，你会对代码和硬件间的交互对程序性能的影响有更敏锐的认识。

这个实验分为 3 个 Part，每个部分单独提交。

1. **Part A** 是一个练手的题目，用来熟悉 Y86-64 工具。
2. **Part B** 允许你对 SEQ simulator 扩展一个新的指令。 
3. **Part C** 是这个 Lab 的核心，前两个 Part 都是为这个 Part 做准备的，在这个 Part 中来优化 Lab 提供的 Y86-64 **benchmark program** 和处理器设计。

___

## 2 Handout 介绍

相关的 Handout 点击[这里](http://csapp.cs.cmu.edu/3e/labs.html)下载。

下载后将文件上传到 Linux 服务器，然后使用命令 `tar xvf archlab-handout.tar` 解压，解压后的文件有 README、Makefile、sim.tar、archlab.pdf 和 simguide.pdf。

然后使用命令 `tar xvf sim.tar`，解压后就会得到上面所说的 Y86-64 工具。这个实验所有的工作都在这个文件夹里完成。

最后执行下面的命令来构建 Y86-64 工具。

```
unix> cd sim
unix> make clean
unix> make
```

在 make 过程中遇到了下面的问题。

```
(cd misc; make all)
make[1]: Entering directory `/usr/csapp/archlab-handout/sim/misc'
gcc -Wall -O1 -g -c yis.c
gcc -Wall -O1 -g -c isa.c
gcc -Wall -O1 -g yis.o isa.o -o yis
gcc -Wall -O1 -g -c yas.c
flex yas-grammar.lex
mv lex.yy.c yas-grammar.c
gcc -O1 -c yas-grammar.c
gcc -Wall -O1 -g yas-grammar.o yas.o isa.o -lfl -o yas
/usr/bin/ld: cannot find -lfl
collect2: error: ld returned 1 exit status
make[1]: *** [yas] Error 1
make[1]: Leaving directory `/usr/csapp/archlab-handout/sim/misc'
make: *** [all] Error 2

```

对于报错 `/usr/bin/ld: cannot find -lfl` 在网上找了一个解决方案。

执行命令 `yum install flex-devel.x86_64`，解决。

紧接着执行 `make` 后，又出现下面的问题：

```
/usr/bin/ld: cannot find -ltk
/usr/bin/ld: cannot find -ltcl
```

执行 `yum install tk-devel.x86_64` 可解。

___

## 3 Part A

这个部分在 sim/misc 这个文件夹里完成。你的任务就是写 3 个 Y86-64 程序并且模拟它。这 3 个程序要实现的功能在 sim/misc/examples.c 里面。

```c
/* 
 * Architecture Lab: Part A
 * 
 * High level specs for the functions that the students will rewrite
 * in Y86-64 assembly language
 */
/* $begin examples */
/* linked list element */
typedef struct ELE {
    long val;
    struct ELE *next;
} *list_ptr;

/* sum_list - Sum the elements of a linked list */
long sum_list(list_ptr ls)
{
    long val = 0;
    while (ls) {
        val += ls->val;
        ls = ls->next;
    }
    return val;
}

/* rsum_list - Recursive version of sum_list */
long rsum_list(list_ptr ls)
{
    if (!ls)
        return 0;
    else {
        long val = ls->val;
        long rest = rsum_list(ls->next);
        return val + rest;
    }
}
/* copy_block - Copy src to dest and return xor checksum of src */
long copy_block(long *src, long *dest, long len)
{
    long result = 0;
    while (len > 0) {
        long val = *src++;
        *dest++ = val;
        result ^= val;
        len--;
    }
    return result;
}
/* $end examples */

```

使用 YAS 将相应的程序转换成二进制，然后再把生成的二进制放到指令集模拟器 YIS 上运行。

### sum.ys: Iteratively sum linked list elements

写一个 Y86-64 程序 `sum.ys` 来计算链表所有元素的和。下面是 3 组链表元素。

```
# Sample linked list
        .align 8
ele1:
        .quad 0x00a
        .quad ele2
ele2:
        .quad 0x0b0
        .quad ele3
ele3:
        .quad 0xc00
        .quad 0
```

这个程序应该先设置栈结构，调用函数，最后 Halt。对于这个问题的解法，可以参考书中的例子 Figure4.7。

```
# Execution begins at address 0
        .pos 0
        irmovq stack, %rsp # Set up stack pointer
        call main          # Execute main program
        halt               # Terminate program

# Array of 4 elements
        .align 8
array:
        .quad 0x000d000d000d
        .quad 0x00c000c000c0
        .quad 0x0b000b000b00
        .quad 0xa000a000a000

main:
        irmovq array,%rdi
        irmovq $4,%rsi
        call sum # sum(array, 4)
        ret

# long sum(long *start, long count)
# start in %rdi, count in %rsi
sum:
        irmovq $8,%r8      # Constant 8
        irmovq $1,%r9      # Constant 1
        xorq %rax,%rax     # sum = 0
        andq %rsi,%rsi     # Set CC
        jmp test           # Goto test
loop:
        mrmovq (%rdi),%r10 # Get *start
        addq %r10,%rax     # Add to sum
        addq %r8,%rdi      # start++
        subq %r9,%rsi      # count--. Set CC
test:
        jne loop           # Stop when 0
        ret # Return

# Stack starts here and grows to lower addresses
        .pos 0x200
stack:
```

下面附上 `sum.ys` 对应的 C 代码和汇编代码。

```c
/* sum_list - Sum the elements of a linked list */
typedef struct ELE {
    long val;
    struct ELE *next;
} *list_ptr;
long sum_list(list_ptr ls)
{
    long val = 0;
    while (ls) {
        val += ls->val;
        ls = ls->next;
    }
    return val;
}
```

```
# Execution begins at address 0
        .pos 0
        irmovq stack, %rsp # Set up stack pointer
        call main          # Execute main program
        halt               # Terminate program

# Sample linked list
        .align 8
ele1:
        .quad 0x00a
        .quad ele2
ele2:
        .quad 0x0b0
        .quad ele3
ele3:
        .quad 0xc00
        .quad 0


main:
        irmovq ele1, %rdi   # 用 %di 来传递参数
        call sum_list
        ret

# long sum_list(list_ptr ls)
# ls store in %rdi

sum_list:
        irmovq $0, %rax     # 将 %rax 初始化为 0，来存储元素和

loop:  
        andq %rdi, %rdi     # 设置 condition codes
        je return           # conditional jump

        mrmovq 0(%rdi), %rsi
        addq %rsi, %rax     # 计算和

        mrmovq 8(%rdi), %rsi
        rrmovq %rsi, %rdi   # 将指针(下一个 struct 的地址)放进 %rdi

        andq %rdi, %rdi
        jne loop

return:
        ret

        .pos 0x400          # TODO 这个值怎么确定？
stack:

```

接下来用 YAS 和 YIS 进行汇编并模拟运行。


