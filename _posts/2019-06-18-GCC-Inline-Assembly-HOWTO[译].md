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

**assembler template** 是一些汇编指令，一条或者多条。**output operands** 是输入操作数，用圆括号 **"("** 和 **")"**，多个操作数以逗号隔开。**input operands** 输出操作数同理。操作数的个数通常与 **assembler template** 中的指令有关。

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
int a=10, b;
asm ("movl %1, %%eax; 
      movl %%eax, %0;"
     :"=r"(b)        /* output */
     :"r"(a)         /* input */
     :"%eax"         /* clobbered register */
     );
```

这条语句使 **b=a=10**。

下面是几条注意的点：

- 变量 b 是输出操作数，与寄存器 %0 关联；变量 a 是输入操作数，与寄存器 %1 关联。
- r 是对操作数的约束（更多约束在后面讲）。这里 **"r"** 告诉 GCC 可以使用任何寄存器来存储操作数 a。**"="** 只用来修饰输出操作数，它限制输出操作数是**只写（write-only）**的。
- 在汇编指令中寄存器以 **%%** 作为前缀，而操作数则以 **%** 为前缀。
- 最后 cloberred register %eax 告诉 GCC：%eax 这个寄存器的内容在 asm 中已经被修改。所以 GCC 不会用这个寄存器来存储其它值。

当 "asm" 执行完后，作为输出操作数的变量 b，值发生了改变。在 "asm" 内部对操作数 b 的修改，反应在了外部的 C 语言中。

#### 5.1、汇编模板

汇编模板是一个汇编指令集合。要么每条指令分别使用双引号，或者所有指令使用一对双引号。每条指令必须以换行符 **\n** 结束，换行符 **\n** 后面可以选择性的跟一个制表符 **\t**。与 C 语言中变量对应的操作数用 %1,%2...表示。

#### 5.2、操作数

C 语言中的表达式作为操作数应用在 "asm" 的汇编指令中。每个操作数前面都有一个双引号括起来的操作数约束。对于输出操作数来说，还会有一个额外的修饰符。

通用的格式就是：

```
"constraint"(C expression)
```

约束主要用来限制操作数的**寻址模式**和指定使用的寄存器。

多个操作数用逗号隔开。

在汇编模板中，每一个操作数被一个数字引用。例如，共有 n 个操作数（包括输入和输出），第一个输出操作数对应数字 0，以此递增，最后一个输入操作数对应数字 n-1。

例3（在例2上稍加修改）：

```
int a=10, b, c=10;
asm ("movl %1, %%eax;
      addl %2, %%eax;
      movl %%eax, %0;"
     :"=r"(b)           /* output */
     :"r"(a), "r"(c)    /* input */
     :"%eax"            /* clobbered register */
     );
```

例 3 中将计算了 **b=a+c** 操作数 b 对应数字 0，a 对应数字 1，c 对应数字 2。

输出操作数必须是 C 语言中的左值表达式，输入操作数则没有这样的限制。扩展内联汇编通常用在编译器都不会优化到的机器指令上。如果输出操作数无法直接寻址（i.e., it's a bit-field），那就必须为它分配一个寄存器。这种情况下，GCC 使用分配的寄存器作为 "asm" 的输出，然后将寄存器的内容放到最终输出中。

前面说过，普通的输出操作数必须是只写（write-only，使用 **=** 修饰符）的。GCC 假设在指令执行前，输出操作数的里的值是无效的。扩展汇编也支持**输入-输出**、**读-写**操作数。

下面来集中看几个例子：

例4：

```
asm ("leal (%1,%1,4), %0"
    : "=r" (five_times_x)
    : "r" (x) 
    );
```

这条 "asm" 计算输入操作数 x 的 5 倍，并将结果放进输出操作数 five_times_x 中。对于输入操作数 x，没有在汇编中指定用哪个寄存器来存它，GCC 会帮我们指定相应的寄存器。我们可以指定 GCC，让输出操作数和输入操作数都使用同一个寄存器：

例5：

```
asm ("leal (%0,%0,4), %0"
     : "=r" (five_times_x)
     : "0" (x) 
     );
```

现在输入和输出操作数都使用了同一个寄存器（因为使用了 **"0"** 匹配约束），但是无法知道是哪个寄存器。通过下面这种方式可以指定寄存器：

例6：

```
asm ("leal (%%ecx,%%ecx,4), %%ecx"
     : "=c" (x)
     : "c" (x) 
     );
```

在上面的 3 个例子中，我们没有在 cloberred list 中指定任何寄存器。在例 4、5 中，是由 GCC 来决定使用哪些寄存器，所以它知道哪些寄存器发生了变化。例 6，我们没有把 ecx 放进 cloberred list，但 GCC 知道它表示 x，既然如此，自然就不用多此一举了。

#### 5.3、Cloberred List

一些指令会破坏寄存器的内容，需要将被破坏的寄存器放进 cloberred list。这是在告诉 GCC：这些寄存器的内容会被修改！这样 GCC 就知道自己向这些寄存器里存放的值已经无效（改变）了。注意，与输出操作数和输入操作数相关联的寄存器不用放进 cloberred list（因为已经通过约束的方式显示的告诉GCC了）。

如果指令会改变条件码寄存器，则需要在 cloberred lis 中添加 **"cc"**。如果指令修改了内存中的内容，则需要在 cloberred lis 中添加 **"memory"**，这样的话 GCC 就不会在寄存器中缓存这块内存的值。如果被影响到的内存没有在输入输出中列出来，也需要加上 **volatile** 关键字。

例7：

```
asm ("movl %0,%%eax;
      movl %1,%%ecx;
      call _foo"
     : /* no outputs */
     : "g" (from), "g" (to)
     : "eax", "ecx"
     );
```

函数 _foo 接受两个指定的参数。

#### 5.4、Volatile

如果你熟悉内核源码或者读过一些很精彩的代码，那么你一定见过很多函数声明带有关键字 **voaltile** 或者 **_voaltile_**。那什么是 **volatile** 呢?

如果你不想你写的汇编语句被优化（例如移动，删除等），那就在 **asm** 关键字后加上 **volatile** 关键字（小心使用）：

```
asm volatile(... : ... : ... : ...);
```

如果我们的汇编只是做一些计算，不会产生副作用，那么最好不要使用 **volatile** 关键字（这样 GCC 才能有效的优化代码）。

&nbsp;

### 六、约束

___

约束能够决定一个操作数在寄存器或者内存，在哪个寄存器或是哪块内存；决定操作数是一个立即数还是一个范围数，等等。

#### 6.1、普通约束

下面是一些常用的约束：

#####6.1.1 寄存器操作数约束（"r"）

当操作数使用这种约束的时候，它（操作数）会被存进通用寄存器（Gnenral Purpose Register, GPR），像这样：

```
asm ("movl %%eax, %0\n" :"=r"(myval));
```

变量 **myval** 被存进一个通用寄存器，再将 %eax 中的值拷贝进这个寄存器。**"r"** 约束表示变量 **myval** 可以存进任何一个通用寄存器。如果想指定一个通用寄存器，需要使用通用寄存器对应的符号：

```
+---+--------------------+
| r |    Random GPR      |
+---+--------------------+
| a |   %eax, %ax, %al   |
| b |   %ebx, %bx, %bl   |
| c |   %ecx, %cx, %cl   |
| d |   %edx, %dx, %dl   |
| S |   %esi, %si        |
| D |   %edi, %di        |
+---+--------------------+
```

#####6.1.2 内存操作数约束（"m"）

当操作数在内存中，就会发生内存引用（as opposed to register constraints, which first store the value in a register to be modified and then write it back to the memory location）。一般来说，只有在必须使用寄存器约束或者能够快程序运行的地方，才会用到寄存器约束。

当你不想用寄存器去保存一个需要改变的 C 变量时，使用内存约束是最好的办法。

例8：

```
asm("sidt %0\n" 
    : 
    :"m"(loc)
);
```

这条语句将 IDTR 寄存器的值，保存在内存位置 loc 处。

#####6.1.3 匹配约束（"数字"）

有时候，一个变量既要当做输入操作数又要当做输出操作数。这种情况下，在 asm 中使用匹配约束来指定。

例9：

```
asm ("incl %0" 
     :"=a"(var)
     :"0"(var)
);
```

这个知识点我们在例4、5、6中有提到过。这里使用匹配约束，寄存器 %eax 既用来保存输入操作数，也用来保存输出操作数。输入变量 var 被存进 %eax，在 %eax 中完成自增。**"0"** 在这里表示与第 0 号输入操作数有相同的约束（既 **"a"**）。这种约束可以用在：

- 从一个变量读取输入，修改后结果存进同一个变量的情况
- 输入操作数和输出操作数没必要分开的情况。

匹配约束可以有效的利用寄存器。

#####6.1.3 匹配约束（"数字"）

其他一些有用的约束：

- "m" : A memory operand is allowed, with any kind of address that the machine supports in general.
- "o" : A memory operand is allowed, but only if the address is offsettable. ie, adding a small offset to the address gives a valid address.
- "V" : A memory operand that is not offsettable. In other words, anything that would fit the 'm' constraint but not the 'o' constraint.
- "i" : An immediate integer operand (one with constant value) is allowed. This includes symbolic constants whose values will be known only at assembly time.
- "n" : An immediate integer operand with a known numeric value is allowed. Many systems cannot support assembly-time constants for operands less than a word wide. Constraints for these operands should use 'n' rather than 'i'.
- "g" : Any register, memory or immediate integer operand is allowed, except for registers that are not general registers.
Following constraints are x86 specific.

- "r" : Register operand constraint, look table given above.
- "q" : Registers a, b, c or d.
- "I" : Constant in range 0 to 31 (for 32-bit shifts).
- "J" : Constant in range 0 to 63 (for 64-bit shifts).
- "K" : 0xff.
- "L" : 0xffff.
- "M" : 0, 1, 2, or 3 (shifts for lea instruction).
- "N" : Constant in range 0 to 255 (for out instruction).
- "f" : Floating point register
- "t" : First (top of stack) floating point register
- "u" : Second floating point register
- "A" : Specifies the 'a' or 'd' registers. This is primarily useful for 64-bit integer values intended to be returned with the 'd' register holding the most significant bits and the 'a' register holding the least significant bits.

#### 6.2、约束修饰符

在使用约束的时候，为了更精确的表述，GCC 提供了一些修饰符：

- "=": 被修饰的操作数是只写的。
- "&": 被修饰的操作数之前被修改过。which is modified before the instruction is finished using the input operands. Therefore, this operand may not lie in a register that is used as an input operand or as part of any memory address. An input operand can be tied to an earlyclobber operand if its only use as an input occurs before the early result is written.

&nbsp;

### 七、实例

___

```
int main(void)
{
    int foo = 10, bar = 15;
    __asm__ __volatile__("addl  %%ebx,%%eax"
                         :"=a"(foo)
                         :"a"(foo), "b"(bar)
                         );
    printf("foo+bar=%d\n", foo);
    return 0;
}
```

这个例子里，我们让 GCC 把 foo 存在 eax 里，bar 存在 ebx 里，和存放在 eax 里。

```
 __asm__ __volatile__(
                      "   lock       ;\n"
                      "   addl %1,%0 ;\n"
                      : "=m"  (my_var)
                      : "ir"  (my_int), "m" (my_var)
                      : /* no clobber-list */
                      );
```

这是一个原子加法。移除 lock 指令，就能移除原子性。"=m" 表明 my_var 是一个输出操作数，在内存中（而不是寄存器中）。"ir" 说明 my_int 是一个整数，并且它应该保存在寄存器中。

```
 __asm__ __volatile__(  "decl %0; sete %1"
                      : "=m" (my_var), "=q" (cond)
                      : "m" (my_var) 
                      : "memory"
                      );
```

my_var 的值自减 1，如果减到 0，cond 设为 1。注意：

- my_var 变量在内存中
- cond 是 eax，ebx，ecx，edx 中的任一个
- 内存可能被修改，所以 clobberred list 中为 "memory"

```
__asm__ __volatile__(   "btsl %1,%0"
                      : "=m" (ADDR)
                      : "Ir" (pos)
                      : "cc"
                      );
```

将内存地址 ADDR 处的变量的 pos 位设为 1（btsl 与 btrl 作用相反）。约束 "Ir" 表明 pos 在寄存器中，范围在 0~31 之间。由于位运算会改变条件码寄存器，所以将 cc 放进 cloberred list。

```
static inline char * strcpy(char * dest, const char *src)
{
int d0, d1, d2;
__asm__ __volatile__(  "1:\tlodsb\n\t"
                       "stosb\n\t"
                       "testb %%al,%%al\n\t"
                       "jne 1b"
                     : "=&S" (d0), "=&D" (d1), "=&a" (d2)
                     : "0" (src),"1" (dest) 
                     : "memory");
return dest;
}
```

前面讲的了，src 的约束为 "0"，表明它与第 0 号输入操作数有相同的约束，即 src 存放在 esi 寄存器。同理，dest 存放在 edi 寄存器。"&S", "&D", "&a" 表明寄存器 esi，edi，eax 是 early cloberred register，在函数完成之前，它们里面的内容会不断变化。

```
#define mov_blk(src, dest, numwords) \
__asm__ __volatile__ (                                          \
                       "cld\n\t"                                \
                       "rep\n\t"                                \
                       "movsl"                                  \
                       :                                        \
                       : "S" (src), "D" (dest), "c" (numwords)  \
                       : "%ecx", "%esi", "%edi"                 \
                       )
```

注意这里我们把内联汇编定义成了宏，这是 linux kernel 中经常用的技巧。

```
#define _syscall3(type,name,type1,arg1,type2,arg2,type3,arg3) \
type name(type1 arg1,type2 arg2,type3 arg3) \
{ \
long __res; \
__asm__ volatile (  "int $0x80" \
                  : "=a" (__res) \
                  : "0" (__NR_##name),"b" ((long)(arg1)),"c" ((long)(arg2)), \
                    "d" ((long)(arg3))); \
__syscall_return(type,__res); \
}
```

在 linux 中，有些系统调用用内联汇编实现，查看 linux/unistd.h 可以看到，所有的系统调用都被定义成了宏，如上。系统调用号存放在 eax 中，参数分别存放在 ebx，ecx 和 edx 中，一切准备就绪后，执行 int 0x80 启动系统调用，最后的返回值存放在 eax 中。所有的系统调用的实现都类似， exit() 是有一个形参的系统调用：

```
{
    asm("movl $1,%%eax;     /* SYS_exit is 1 */
         xorl %%ebx,%%ebx;  /* Argument is in ebx, it is 0 */
         int  $0x80"        /* Enter kernel mode */
         );
}
```

exit() 是第 1 号系统调用，参数为 0。

&nbsp;

### 八、总结

___

本片文章主要讲解 GCC 内联汇编，了解基本概念后就可以自己摸索了。

GCC 内联是一个很强大的功能，本文也只是介绍了点基本概念。想要更权威的讲解，可以参考：

[6.47 How to Use Inline Assembly Language in C Code](https://gcc.gnu.org/onlinedocs/gcc/Using-Assembly-Language-with-C.html#Using-Assembly-Language-with-C)

众所周知，linux 内核中大量的使用了内联汇编，想要更深入学习的也可以参考 linux 内核源码。