---
layout: post
title: 'Systemtap tutorial[译]'
subtitle: ''
date: 2019-12-21
categories: 技术
cover: ''
tags: Linux
---

本文译自：[Systemtap tutorial](https://sourceware.org/systemtap/tutorial.pdf)

最近一年里，读了一两本内核方面的书籍。由于自己不是从事内核方面的工作，相关知识大多一学就忘。为了巩固学到的知识，用兴趣驱动学习，准备从Linux里面的一个性能优化工具入手，这就是大名鼎鼎的Systemtap。下面是Systemtap的官方入门教程翻译：

# 1 Introduction

> Systemtap is a tool that allows developers and administrators to write and reuse simple scripts to deeply examine the activities of a live Linux system. Data may be extracted, filtered, and summarized quickly and safely, to enable diagnoses of complex performance or functional problems.

Systemtap允许开发者和管理员编写并重用简单的脚本，这些脚本可以深度检查运行中的Linux系统。我们可以快速安全地提取、过滤并总结由Systemtap得到的数据，以此来诊断复杂的性能问题。

注意：本教程不会详细介绍到Systemtap的每个细节，详情请参考：[Systemtap manual pages](https://sourceware.org/systemtap/man/)。

> The essential idea behind a systemtap script is to name events, and to give them handlers. Whenever a specified event occurs, the Linux kernel runs the handler as if it were a quick subroutine, then resumes. There are several kind of events, such as entering or exiting a function , a timer expiring, or the entire systemtap session starting or stopping. A handler is a series of script language statements that specify the work to be done whenever the event occurs. This work normally includes extracting data from the event context, storing them into internal variables, or printing results.

Systemtap脚本背后的关键概念是**event**，以及与event相关联的**handler**。一旦一个指定的event发生，Linux会去执行与event相关联的handler，就像执行一个子例程，执行完后恢复。这里有几种event，例如：进入或退出一个函数、一个计时器过期、Sytemtap会话开始或停止。Handler其实是一系列的脚本语句，这些脚本语句指明了event发生后要完成的一些工作，这些工作包括从event上下文提取数据，并将数据保存下来或打印出来。

> Systemtap works by translating the script to C, running the system C compiler to create a kernel module from that. When the module is loaded, it activates all the probed events by hooking into the kernel. Then, as events occur on any processor, the compiled handlers run. Eventually, the session stops, the hooks are disconnected, and the module removed. This entire process is driven from a single command-line program, stap.

Systemtap的工作是将脚本翻译成C语言的形式，将翻译后的代码编译成内核模块。当模块加载到内核时，它会被hook进内核，以此来激活所有可探测的（probed）event。接着，event会发生在任何处理器上，编译好的handler开始执行。最终，Systemtap会话停止，模块与内核断开并被移除。整个过程由一个简单的命令行程序驱动：**stap**。

```sh
# cat hello-world.stp
probe begin
{
    print ("hello world\n")
    exit ()
}

# stap hello-world.stp
hello world
```

> This paper assumes that you have installed systemtap and its prerequisite kernel development tools and debugging data, so that you can run the scripts such as the simple one in Figure 1. Log on as root, or even better, login as a user that is a member of stapdev group or as a user authorized to sudo, before running systemtap. 

略。

# 2 Tracing

> The simplest kind of probe is simply to trace an event. This is the effect of inserting strategically located print statements into a program. This is often the first step of problem solving: explore by seeing a history of what has happened.

最简单的探测是追踪一个event。这是将策略定位打印语句插入到程序中的效果。解决问题的第一步通常是：通过观察过去发生的事情来探索。

```sh
# cat strace-open.stp
probe syscall.open
{
    printf ("%s(%d) open (%s)\n", execname(), pid(), argstr)
}
probe timer.ms(4000) # after 4 seconds
{
    exit ()
}

# stap strace-open.stp
vmware-guestd(2206) open ("/etc/redhat-release", O_RDONLY)
hald(2360) open ("/dev/hdc", O_RDONLY|O_EXCL|O_NONBLOCK)
hald(2360) open ("/dev/hdc", O_RDONLY|O_EXCL|O_NONBLOCK)
hald(2360) open ("/dev/hdc", O_RDONLY|O_EXCL|O_NONBLOCK)
df(3433) open ("/etc/ld.so.cache", O_RDONLY)
df(3433) open ("/lib/tls/libc.so.6", O_RDONLY)
df(3433) open ("/etc/mtab", O_RDONLY)
hald(2360) open ("/dev/hdc", O_RDONLY|O_EXCL|O_NONBLOCK)
```

>This style of instrumentation is the simplest. It just asks systemtap to print something at each event. To express this in the script language, you need to say where to probe and what to print there.

这种测量是最简单的，仅仅是用Systemtap在每个event处打印一些东西。为了用脚本语言来表述，你需要（1）指明在哪里探测和（2）在探测点打印什么。

## 2.1 Where to probe

> Systemtap supports a number of built-in events. The library of scripts that comes with systemtap,each called a “tapset”, may define additional ones defined in terms of the built-in family. See the stapprobes man page for details on these and many other probe point families. All these events are named using a unified syntax with dot-separated parameterized identifiers:

Systemtap支持一些内置event。Systemtap随附的脚本库（每个脚本库称为”tapset“）可以根据内置event定义的其他脚本。有关这些以及其他许多探测点的详细信息，请参见**stapprobes**手册页。所有这些events均使用**点分**的参数化标识符的统一语法来命名：

- `begin`：The startup of the systemtap session.
- `end`：The end of the systemtap session.
- `kernel.function("sys_open")`：The entry to the function named sys_open in the kernel.
- `syscall.close.return`：The return from the close system call.
- `module("ext3").statement(0xdeadbeef)`： The addressed instruction in the ext3 filesystem driver.
- `timer.ms(200)`：A timer that fires every 200 milliseconds.
- `timer.profile`：A timer that fires periodically on every CPU.
- `perf.hw.cache_misses`：A particular number of CPU cache misses have occurred.
- `procfs("status").read`：A process trying to read a synthetic file.
- `process("a.out").statement("*@main.c:200")`：Line 200 of the a.out program.

> Let’s say that you would like to trace all function entries and exits in a source file, say net/socket.c in the kernel. The kernel.function probe point lets you express that easily, since systemtap examines the kernel’s debugging information to relate object code to source code. It works like a debugger: if you can name or place it, you can probe it. Use kernel.function("*@net/socket.c").call for the function entries, and kernel.function("*@net/socket.c").return for matching exits. Note the use of wildcards in the function name part, and the subsequent @FILENAME part. You can also put wildcards into the file name, and even add a colon (:) and a line number, if you want to restrict the search that precisely. Since systemtap will put a separate probe in every place that matches a probe point, a few wildcards can expand to hundreds or thousands of probes, so be careful what you ask for.

假设你想跟踪一个源文件里所有函数的入口和出口，例如内核中的net/socket.c。探测点`kernel.function`可以很轻松的做到，因为Systemtap会检查内核的调试信息已将目标代码与源代码相关联。它像调试器一样工作：如果可以命名或放置它，则可以对其进行探测。分别使用`kernel.function("*@net/socket.c").call`（注意，如果不使用call，内联函数也会被探测到，但是它们没有响应的return）、`kernel.function("*@net/socket.c").return`来匹配函数的入口和出口。注意函数名部分使用了通配符。你也可以在@后面的文件名部分使用通配符，甚至可以在文件名后面加上冒号和行号（译注：例如，kernel.function("FUNCNAME@FILENAME:LINE")）。因为Systemtap会在每个匹配探测点的地方放上一个单独的探测，一个简单的通配符可能会引入成百上千个探测，请谨慎使用。

> Once you identify the probe points, the skeleton of the systemtap script appears. The probe keyword introduces a probe point, or a comma-separated list of them. The following { and } braces enclose the handler for all listed probe points.

一旦确定了探测点，紧接着就是Systemtap脚本框架。**probe**关键字引入了一个探测点，或多个探测点用**点分割**（译注：原文是comma-separated，即逗号分割，根据上下文，我觉得此处应该是dot-separated）而成的列表。

```sh
probe kernel.function("*@net/socket.c") { }
probe kernel.function("*@net/socket.c").return { }
```

> You can run this script as is, though with empty handlers there will be no output. Put the two lines into a new file. Run stap -v FILE. Terminate it any time with ^C. (The -v option tells systemtap to print more verbose messages during its processing. Try the -h option to see more options.)

## 2.2 What to print

> Since you are interested in each function that was entered and exited, a line should be printed for each, containing the function name. In order to make that list easy to read, systemtap should indent the lines so that functions called by other traced functions are nested deeper. To tell each single process apart from any others that may be running concurrently, systemtap should also print the process ID in the line.

> Systemtap provides a variety of such contextual data, ready for formatting. They usually appear as function calls within the handler, like you already saw in Figure 1. See the function::* man pages for those functions and more defined in the tapset library, but here’s a sampling:

- tid()： The id of the current thread.
- pid()： The process (task group) id of the current thread.
- uid()： The id of the current user.
- execname()： The name of the current process.
- cpu()： The current cpu number.
- gettimeofday_s()： Number of seconds since epoch.
- get_cycles()： Snapshot of hardware cycle counter.
- pp()： A string describing the probe point being currently handled.
- ppfunc()： If known, the the function name in which this probe was placed.
- $$vars： If available, a pretty-printed listing of all local variables in scope.
- print_backtrace()： If possible, print a kernel backtrace.
- print_ubacktrace()： If possible, print a user-space backtrace.

> The values returned may be strings or numbers. The print() built-in function accepts either as its sole argument. Or, you can use the C-style printf() built-in, whose formatting argument may include %s for a string, %d for a number. printf and other functions take comma-separated arguments. Don’t forget a "\n" at the end. There exist more printing / formatting functions too.

> A particularly handy function in the tapset library is thread_indent. Given an indentation delta parameter, it stores internally an indentation counter for each thread (tid()), and returns a string with some generic trace data plus an appropriate number of indentation spaces. That generic data includes a timestamp (number of microseconds since the initial indentation for the thread), a process name and the thread id itself. It therefore gives an idea not only about what functions were called, but who called them, and how long they took. Figure 3 shows the finished script. It lacks a call to the exit() function, so you need to interrupt it with ^C when you want the tracing to stop.

```sh
# cat socket-trace.stp
probe kernel.function("*@net/socket.c").call {
    printf ("%s -> %s\n", thread_indent(1), ppfunc())
}
probe kernel.function("*@net/socket.c").return {
    printf ("%s <- %s\n", thread_indent(-1), ppfunc())
}

# stap socket-trace.stp
0 hald(2632): -> sock_poll
28 hald(2632): <- sock_poll
[...]
0 ftp(7223): -> sys_socketcall
1159 ftp(7223): -> sys_socket
2173 ftp(7223): -> __sock_create
2286 ftp(7223): -> sock_alloc_inode
2737 ftp(7223): <- sock_alloc_inode
3349 ftp(7223): -> sock_alloc
3389 ftp(7223): <- sock_alloc
3417 ftp(7223): <- __sock_create
4117 ftp(7223): -> sock_create
4160 ftp(7223): <- sock_create
4301 ftp(7223): -> sock_map_fd
4644 ftp(7223): -> sock_map_file
4699 ftp(7223): <- sock_map_file
4715 ftp(7223): <- sock_map_fd
4732 ftp(7223): <- sys_socket
4775 ftp(7223): <- sys_socketcall
[...]
```

## 2.3 Exercises

1. Use the -L option to systemtap to list all the kernel functions named with the word “nit” in them.
2. Trace some system calls (use syscall.NAME and .return probe points), with the same thread_indent
probe handler as in Figure 3. Print parameters using $$parms and $$return. Interpret the results.
3. Change figure 3 by removing the .call modifier from the first probe. Note how function entry and
function return now don’t match anymore. This is because now the first probe will match both normal
function entry and inlined functions. Try putting the .call modifier back and add another probe just
for probe kernel.function("*@net/socket.c").inline What printf statement can you come up
with in the probe handler to show the inlined function entries nicely in between the .call and .return
thread indented output?

# 3 Analysis

## 3.1 Basic constructs

## 3.2 Target variables

## 3.3 Functions

## 3.4 Arrays

## 3.5 Aggregates

## 3.6 Safety

## 3.7 Exercises

# 4 Tapsets

## 4.1 Automatic selection

## 4.2 Probe point aliases

## 4.3 Embedded C

## 4.4 Naming conventions

## 4.5 Exercises

# 5 Further information

# A Glossary

# B Errors

## B.1 Parse errors

## B.2 Type errors

## B.3 Symbol errors

## B.4 Probing errors

## B.5 Runtime errors

# C Acknowledgments