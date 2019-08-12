---
layout: post
title: 'Go语言汇编器的设计'
subtitle: 'Rob Pike在开发者大会上的演讲'
date: 2019-07-22
categories: 技术
cover: ''
tags: Golang
---

这篇博文来自视频 [GopherCon 2016: Rob Pike - The Design of the Go Assembler](https://www.youtube.com/watch?v=KINIAgRpkDA) 里，Rob Pike 关于 Go 汇编器的讲述，记录部分我认为比较重要的文字。下面视频来自 youtube:

<div class="video-container"><iframe src="https://www.youtube.com/embed/KINIAgRpkDA" frameborder="0" allowfullscreen></iframe></div>

> The most important single thing to realize about assembler language is that it enables the programmer to use all System/360 machine functions as if he were coding in System/360 machine language. - A programmer's Intruduction to IBM System/360 Assembler Language, 1970

> The assembler is how you talk to the machine at very lowest level.

> CPUs look pretty much the same nowadays, most of them(certainly all the ones that go runs on) are pretty much the same if you ignore a lot of detail. We can construct a common grammer for all of these machines.

接着 Rob 介绍了 plan 9 上的 C compiler。

> The plan 9 C compiler didn't actually generate real instructions, it generated what you might think of as pseudo-code. Then the second program that run on the output of the compiler which is the linker actually turned those pseudo instructions into real instructions.

可见在 plan 9 上，没有 assembler 这个概念。Compiler 输出伪码到 linker，如果写一句如下伪码:

```
MOVW $0, var
```

plan 9 上的 linker 可能会将上面的伪码翻译成下面这样:

```
XORW  R1, R1
STORE R1, var
```

这个过程称为 [Instrction selection](https://en.wikipedia.org/wiki/Instruction_selection)。

![](http://ww1.sinaimg.cn/large/c9caade4gy1g58juhzonlj21840k6q7q.jpg)

> The top line is the sort of tranditioal architecture, like GCC runs. The bottom two are what the plan 9 architecture look like. The assembler was cut half and the first part of the assembler got stuck into the compiler and the second part of the assembler went into the linker, what flowing across the red line is kind of binary represtation of the pseudo instructions.

> All the assembler is doing in plan 9 world is giving a way to write textual version of instruction that drive the linker.

> The plan 9 world has a compiler drive the linker or the assembler drive the linker.

> In Go1.3, Russ took a big chunk of linker and pulled it out, and make it a library called "liblink", a big piece of which was what we called [Instrction selection](https://en.wikipedia.org/wiki/Instruction_selection) algorithm.

现在 liblink 这个库改名为 [obj](https://golang.org/pkg/cmd/internal/obj/#pkg-overview)。

> The compiler use liblink to generate the real instruction from the pseudo instruction.

将原本 linker 做的事转移到 compiler 中了，这样做加快了 build 的速度(因为compiler承担了更多的工作)。

> The compiler slows down but the builds actually sped up.

由于 instruction selection 的重新设计，整个流程变成了这样:

![](http://ww1.sinaimg.cn/large/c9caade4gy1g58ku3q9xij21840k6afj.jpg)

> After Go1.5, the many compiler(6g, 8g etc.) were replaced with a single tool: compile.

