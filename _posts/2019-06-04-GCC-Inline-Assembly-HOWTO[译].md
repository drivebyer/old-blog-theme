---
layout: post
title: "GCC Inline Assembly HOWTO[译]"
comments: true
description: "GCC Inline Assembly HOWTO[译]"
keywords: "inline assembly, c"
---

由于最近在完成 JOS 的 lab3，需要用到不少的内联汇编的知识，准备集中学习一下，将其中一篇讲得比较好的文章翻译下来，在译文的基础上删掉一些我认为没必要的也添加了一些自己的理解。

原文链接：[GCC-Inline-Assembly-HOWTO](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html)

[TOC]

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

### 三、语法

___

