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

使用 YAS 将相应的程序转换成二进制，然后再把生成的二进制放到指令集模拟器 YIS 上运行。

### sum.ys: 计算链表元素和

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

下面附上 sum.ys 对应的 C 代码和汇编代码。

```c
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
        irmovq stack, %rsp # 设置 stack pointer
        call main          
        halt               
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
        irmovq ele1, %rdi    # 用 %di 来传递参数
        call sum_list
        ret
# long sum_list(list_ptr ls)
# ls store in %rdi
sum_list:
        irmovq $0, %rax      # 将 %rax 初始化为 0，来存储元素和
loop:  
        andq %rdi, %rdi      # 设置 condition codes
        je return            # conditional jump
        mrmovq 0(%rdi), %rsi
        addq %rsi, %rax      # 计算和
        mrmovq 8(%rdi), %rdi # 将指针(下一个 struct 的地址)放进 %rdi
        andq %rdi, %rdi
        jne loop
return:
        ret
        .pos 0x400           # TODO 这个值怎么确定？
stack:

```

接下来用 YAS 和 YIS 进行汇编并模拟运行。

![sum.yo模拟执行](http://ww1.sinaimg.cn/large/c9caade4ly1fzhyk4586xj20ck04ndfr.jpg)

可以看到 %rax 的值就是标号 ele1，ele2，ele3 处三个元素的和 0xcba，并且可以看到部分寄存器和部分内存地址的值也发生了改变（TODO 内存的值为什么会改变）。

### rsum.ys: 递归计算链表元素和

写一个类似的 Y86-64 程序 rsum.ys 递归的计算链表的和，链表元素与上面一样。

下面附上 rsum.ys 对应的 C 代码和汇编代码。

```c
typedef struct ELE {
    long val;
    struct ELE *next;
} *list_ptr;
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
```

```
# Execution begins at address 0
        .pos 0
        irmovq stack, %rsp # 设置 stack pointer
        call main          
        halt               
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
        call rsum_list
        ret
# long rsum_list(list_ptr ls)
# ls store in %rdi
rsum_list:
        pushq %r12           # 这个寄存器称为 callee-save registers
                             # 在这个函数里将要使用到 %r12，所以将里面原有的值保存起来
        irmovq $0, %rax      # 将 %rax 初始化为 0，来存储元素和
        andq %rdi, %rdi      # 设置 condition codes
        je return            # conditional jump
        mrmovq 0(%rdi), %r12 # 将当前 struct 的 val 值放进寄存器
        mrmovq 8(%rdi), %rdi # 加载下一个 struct 的地址到寄存器
        call rsum_list       # 前一个函数还没弹栈，下一个函数接着压栈 (TODO 递归调用的栈帧是什么样的？)
        addq %r12, %rax      # 计算和
return:
        popq %r12            # 函数调用结束时记得从栈中恢复这个寄存器原来的值
        ret
        .pos 0x400           # TODO 这个值怎么确定？
stack:

```

接下来用 YAS 和 YIS 进行汇编并模拟运行。

![rsum.yo模拟执行](http://ww1.sinaimg.cn/large/c9caade4ly1fzi2us3v3cj20cp07bq2y.jpg)

通过分析该函数的汇编代码，可以很容易的弄清递归的调用方式。`rsum_list()` 函数每调用一次，就将当前 struct 的 **val** 值通过 %r12 压入当下函数的栈帧中，并且将 **next** 指针值通过 %rdi 传递给下一个函数 `rsum_list()`。

当到达最后一个 struct 时，执行 `je return` 开始弹栈。每次弹栈之前，就从栈中恢复一个之前压入栈中的 **val** 值到 %r12 中，然后执行 `ret` 回到上一次调用 `call rsum_list` 的后面 `addq %r12, %rax` 进行元素的加法累计运算，直到最后函数全部弹栈完成，%rax 中存储了最终的元素和。

TODO 但是通过 [Y86 Simulator Seb](https://github.com/quietshu/y86) 来可视化执行上面的 `rsum.yo` 发现：函数第一次执行 Conditional Jump `je return` 就跳转到了 **return** 标号处，这与代码逻辑不相符。

### copy.ys: 拷贝函数

将内存中的一块数据拷贝到另一个不重叠的内存位置，并计算被拷贝数据的 **checksum(Xor)**。C 语言代码如下：

```c
typedef struct ELE {
    long val;
    struct ELE *next;
} *list_ptr;
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
```

下面是汇编代码：

```
# Execution begins at address 0
        .pos 0
        irmovq stack, %rsp
        call main
        halt
.align 8
# Source block
src:
        .quad 0x00a
        .quad 0x0b0
        .quad 0xc00
# Destination block
dest:
        .quad 0x111
        .quad 0x222
        .quad 0x333
# main method
main:
        irmovq src, %rdi
        irmovq dest, %rsi
        irmovq $3, %rdx      # 调用函数之前准备好参数
        call copy_block
        ret
# long copy_block(long* src, long* dest, long len)
copy_block:
        pushq %r12           # 临时存放 val 值
        irmovq $0, %rax      # result = 0，TODO %rax 中的值不需要保存吗？
        jmp loop_test
loop:  
        mrmovq 0(%rdi), %r12 # long val = *src
        addq $8, %rdi        # src++  
        rmmovq %r12, (%rsi)  # *dest = val
        addq $8, %rsi        # dest++，这样 dest 中就存放了下一个 struct 的地址
        xorq %r12, %rax      # result ^= val
        subq $1, %rdx        # len--
        jmp 
loop_test:
        andq %rdx, %rdx      # len > 0？
        jg loop
        popq %r12
        ret
        .pos 0x300
stack:

```

将代码上传至服务器，执行 `./yas copy.ys` 出现下面的问题：

![不能直接对立即数计算](http://ww1.sinaimg.cn/large/c9caade4ly1fzm8q9j7gbj20jw03v74b.jpg)

看样子是因为 Y86-64 里不支持立即数的运算，这个知识点应该在第三章讲过，一段时间没碰汇编给忘了。

为了解决这个问题，再次使用两个 callee-register 来保存 1 和 8 这两个数，函数部分修改后如下：

```
copy_block:
        pushq %r12           # 临时存放 val 值
        pushq %r13           #
        pushq %r14           #
        irmovq $0, %rax      # result = 0，TODO %rax 中的值不需要保存吗？
        irmovq $1, %r13      #
        irmovq $8, %r14      #
        jmp loop_test
loop:  
        mrmovq 0(%rdi), %r12 # long val = *src
        addq %r14, %rdi        # src++  
        rmmovq %r12, (%rsi)  # *dest = val
        addq %r14, %rsi        # dest++，这样 dest 中就存放了下一个 struct 的地址
        xorq %r12, %rax      # result ^= val
        subq %r13, %rdx        # len--
loop_test:
        andq %rdx, %rdx      # len > 0？
        jg loop
        popq %r14            # TODO 与 push 时顺序相反
        popq %r13
        popq %r12
        ret
```

编译并模拟运行如下：

![Pop顺序正确](http://ww1.sinaimg.cn/large/c9caade4ly1fzm9kv3qz7j20cp06emx5.jpg)

如果将 popq 逆序，得到的结果如下。

![Pop顺序错误](http://ww1.sinaimg.cn/large/c9caade4ly1fzm9mpbcbkj20cl06taa2.jpg)

TODO 这种操作是错误的，照理来说 %r14 和 %12 都应该发生变化，为什么只有 %r12 里的值变了？

___

## 4 Part B

这部分在目录 **sim/seq** 里完成。

Part B 的任务就是扩展 SEQ Processor，通过修改 seq-full.hcl 文件，使其支持 iaddq 指令。

参照书中 Figure 4.18 中的 OPq 和 irmovq 指令描述。

![Figure4.18](http://ww1.sinaimg.cn/large/c9caade4ly1fzmafgx13kj20il0cjmyb.jpg)

得到 iaddq 指令的五阶段描述：

| Stage | iaddq                     |
| ----- |:-------------------------:|
| Fetch | icode :ifun <- M1[PC]     |
|       |                           |
## 5 Part C