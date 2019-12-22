---
layout: post
title: 'Systemtap tutorial[译]'
subtitle: ''
date: 2019-12-21
categories: 技术
cover: ''
tags: Systemtap
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

一旦确定了探测点，紧接着就是Systemtap脚本框架。**probe**关键字引入了**一个探测点**或逗号分割（译注：多个探测点的情况见下文中inode-watch.stp）的**多个探测点**而成的列表。紧接着的大括号里是对前面所有（点分割）探测点生效的handler。

```sh
probe kernel.function("*@net/socket.c") { }
probe kernel.function("*@net/socket.c").return { }
```

> You can run this script as is, though with empty handlers there will be no output. Put the two lines into a new file. Run stap -v FILE. Terminate it any time with ^C. (The -v option tells systemtap to print more verbose messages during its processing. Try the -h option to see more options.)

如上，虽然大括号内的hanlder为空，但是脚本仍然可以运行。将上面两行放入文件中，输入`stap -v FILE`运行。

## 2.2 What to print

> Since you are interested in each function that was entered and exited, a line should be printed for each, containing the function name. In order to make that list easy to read, systemtap should indent the lines so that functions called by other traced functions are nested deeper. To tell each single process apart from any others that may be running concurrently, systemtap should also print the process ID in the line.

因为你对进入和退出的每个函数都感兴趣，因此应该为每个函数打印一行，其中包含函数名称。为了易于阅读，Systemtap应该使行缩进，以便将其他跟踪函数调用的函数嵌套得更深。为了同时运行的进程区别开来，Systemtap还应该在该行中打印进程ID。

> Systemtap provides a variety of such contextual data, ready for formatting. They usually appear as function calls within the handler, like you already saw in Figure 1. See the function::* man pages for those functions and more defined in the tapset library, but here’s a sampling:

Systemtap提供了一些可以格式化的上下文数据。它们通常是出现在handler中的一些函数调用。在**funciton::\***手册或tapset库中可以查到更多的信息，下面是一些简单的例子：

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

这些函数调用的返回值可能是字符串或者数字。内置函数`print()`只接受其中一种类型的参数，或者可以使用内置的C样式`printf()`，其格式化参数％s表示字符串，％d表示数字。`printf()`和其他函数的参数采用逗号分割，末尾别忘了加上“\n”。还有更多的打印 / 格式化函数。

> A particularly handy function in the tapset library is thread_indent. Given an indentation delta parameter, it stores internally an indentation counter for each thread (tid()), and returns a string with some generic trace data plus an appropriate number of indentation spaces. That generic data includes a timestamp (number of microseconds since the initial indentation for the thread), a process name and the thread id itself. It therefore gives an idea not only about what functions were called, but who called them, and how long they took. Figure 3 shows the finished script. It lacks a call to the exit() function, so you need to interrupt it with ^C when you want the tracing to stop.

tapset库中有一个很方便的函数叫做`thread_indent()`。指定一个缩进参数，它将在内部为每个线程存储一个缩进计数器（`tid()`），并返回一个字符串，其中包含一些通用跟踪数据以及适当数量的缩进空间。该通用数据包括时间戳（自线程初始缩进以来的微秒数），进程名称和线程ID。因此，它不仅给出了关于调用什么函数的信息，还给出了调用方以及花费了多长时间。下面是一个示例脚本，它没有对`exit()`函数的调用，因此当你想停止跟踪时，需要用ctrl+C中断它。

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
[...]
```

译注：第一次见`thread_indent()`这个函数，可能会不知所以，下面贴上这个函数的源码，有精力的同学可以细细品味一下：

```sh
function thread_indent:string (delta:long)
{
    return _generic_indent (tid(), sprintf("%s(%d)", execname(), tid()), delta)
}

global _indent_counters, _indent_timestamps
function _generic_indent (idx, desc, delta)
{
    ts = __indent_timestamp()
    if (! _indent_counters[idx]) _indent_timestamps[idx] = ts

    # pre-increment for positive delta and post-decrement for negative delta
    x = _indent_counters[idx] + (delta > 0 ? delta : 0)
    _indent_counters[idx] += delta

    return sprintf("%6d %s:%-*s", (ts - _indent_timestamps[idx]), desc, (x>0 ? x-1 : 0), "")
}
```

## 2.3 Exercises

略。

# 3 Analysis

> Pages of generic tracing text may give you enough information for exploring a system. With systemtap, it is possible to analyze that data, to filter, aggregate, transform, and summarize it. Different probes can work together to share data. Probe handlers can use a rich set of control constructs to describe algorithms, with a syntax taken roughly from awk. With these tools, systemtap scripts can focus on a specific question and provide a compact response: no grep needed.

跟踪文本可以给你足够的信息来探索一个系统。使用Systemtap，可以对数据进行过滤、汇总、转换和总结。不同的探测可以在一起共享数据。探测的handler可以用一个丰富的控制结构集来描述算法，语法大部分取自awk。使用这些工具，Systemtap脚本可以专注于一个特定的问题，返回一个紧凑的结果，不再需要grep。

## 3.1 Basic constructs

> Most systemtap scripts include conditionals, to limit tracing or other logic to those processes or users or
whatever of interest. The syntax is simple:

大多数Systemtap脚本包括条件，以将跟踪或其他逻辑限制于那些进程或用户或感兴趣的点。语法很简单：

```sh
if (EXPR) STATEMENT [else STATEMENT] if/else statement
while (EXPR) STATEMENT               while loop
for (A; B; C) STATEMENT              for loop
```

> Scripts may use break/continue as in C. Probe handlers can return early using next as in awk. Blocks of
statements are enclosed in { and }. In systemtap, the semicolon (;) is accepted as a null statement rather
than as a statement terminator, so is only rarely（） necessary. Shell-style (#), C-style (/* */), and C++-style
(//) comments are all accepted.

脚本可以和C一样使用**break/continue**关键字。探测handler可以和awk一样使用next关键字来提前返回。语句块使用大括号包裹。在Systemtap，分号被当作null而不是语句终止符，因此很少使用（在连续的表达式间放置+/-，++/--是一种模糊的行为）。以下三种注释在Systemtap脚本中都是接受的：（1）#（2）/* */（3）//

> Expressions look like C or awk, and support the usual operators, precedences, and numeric literals. Strings are treated as atomic values rather than arrays of characters. String concatenation is done with the dot("a" . "b"). Some examples:

handler里的表达式看起来像C或awk，支持常用的操作符、优先级、和数字字面量。字符串被当作原子值而不是字符数组。字符串使用点连接，如下：

```sh
(uid() > 100)                                   probably an ordinary user
(execname() == "sed")                           current process is sed
(cpu() == 0 && gettimeofday_s() > 1140498000)   after Feb. 21, 2006, on CPU 0
"hello" . " " . "world"                         a string in three easy pieces
```

> Variables may be used as well. Just pick a name, assign to it, and use it in expressions. They are automatically
initialized and declared. The type of each identifier – string vs. number – is automatically inferred by
systemtap from the kinds of operators and literals used on it. Any inconsistencies will be reported as errors.
Conversion between string and number types is done through explicit function calls.

也可以使用变量，只需选择一个名称，分配给它，然后在表达式中使用它即可。他们是自动的初始化并声明。每个标识符的类型（字符串还是数字）由在Systemtap中使用时的操作符和字面量自动推断，任何不一致之处都将报告为错误。字符串和数字类型之间的转换是通过显式函数调用完成的。

```sh
foo = gettimeofday_s()              foo is a number
bar = "/usr/bin/" . execname()      bar is a string
c++                                 c is a number
s = sprint(2345)                    s becomes the string ”2345”
```

> By default, variables are local to the probe they are used in. That is, they are initialized, used, and disposed of at each probe handler invocation. To share variables between probes, declare them global anywhere in the script. Because of possible concurrency (multiple probe handlers running on different CPUs), each global variable used by a probe is automatically read- or write-locked while the handler is running.

默认情况下，变量在使用它们的探针上是local的。也就是说，在每次探测handler调用时都将对其进行初始化，使用和处置。要在探测之间共享变量，请在脚本中的任何位置全局声明它们。由于可能存在并发性（多个探针handler在不同的CPU上运行），探针使用的每个全局变量在handler运行时会自动被读或写锁定。

## 3.2 Target variables

> A class of special “target variables” allow access to the probe point context. In a symbolic debugger, when you’re stopped at a breakpoint, you can print values from the program’s context. In systemtap scripts, for those probe points that match with specific executable point (rather than an asynchronous event like a timer), you can do the same.

一类特殊的“目标变量”允许访问探测点的上下文。在符号调试器中，在断点处停止时，可以从程序的上下文中打印值。在Systemtap脚本中，对于与特定可执行点匹配的那些探测点（而不是像计时器这样的异步事件），你可以执行相同的操作。

> In addition, you can take their address (the & operator), pretty-print structures (the $ and $$ suffix), pretty-print multiple variables in scope (the $$vars and related variables), or cast pointers to their types (the @cast operator), or test their existence / resolvability (the @defined operator). Read about these in the manual pages.

另外，你可以获取目标变量（target variable）地址（＆运算符），易打印的结构（$和$$后缀），打印范围内的多个变量（$$var和相关变量），或将指针转换为它们的类型（ @cast运算符），或测试它们的存在/可解析性（@defined运算符）。在手册页中阅读这些内容。

> To know which variables are likely to be available, you will need to be familiar with the kernel source you are probing. In addition, you will need to check that the compiler has not optimized those values into unreachable nonexistence. You can use stap -L PROBEPOINT to enumerate the variables available there.

要知道哪些变量可用，需要熟悉要探测的内核源码。另外，检查编译器有没有将这些值优化为无法访问。可以使用`stap -L PROBEPOINT`枚举在探测点可用的变量。

> Let’s say that you are trying to trace filesystem reads/writes to a particular device/inode. From your knowledge of the kernel, you know that two functions of interest could be vfs_read and vfs_write. Each takes a struct file * argument, inside there is either a struct dentry * or struct path * which has a struct dentry *. The struct dentry * contains a struct inode *, and so on. Systemtap allows limited dereferencing of such pointer chains. Two functions, user_string and kernel_string, can copy char * target variables into systemtap strings. Figure 5 demonstrates one way to monitor a particular file (identified by device number and inode number). The script selects the appropriate variants of dev_nr andinode_nr based on the kernel version. This example also demonstrates passing numeric command-line arguments ($1 etc.) into scripts.

假设你正在尝试跟踪文件系统对特定设备或inode的读/写。根据对内核的了解，你知道感兴趣的两个函数可能是`vfs_read()`和`vfs_write()`。 每个都带有一个`struct file*`类型的参数，里面有一个`struct dentry*`或带有`struct dentry*`的`struct path*`。`struct dentry*`包含一个`struct inode*`，依此类推。 Systemtap允许对此类指针链进行**有限制**的dereferencing。`user_string()`和`kernel_string()`这两个函数可以将`char*`目标变量复制到Systemtap字符串中。下面的脚本演示了一种监视特定文件的方式（由设备号和索引节点号标识）。该脚本根据内核版本选择`dev_nr`和`inode_nr`的适当变体。 此示例还演示了如何将数字命令行参数（$1等）传递到脚本中。

```sh
# cat inode-watch.stp
probe kernel.function ("vfs_write"), kernel.function ("vfs_read")
{
    if (@defined($file->f_path->dentry)) {
        dev_nr = $file->f_path->dentry->d_inode->i_sb->s_dev
        inode_nr = $file->f_path->dentry->d_inode->i_ino
    } else {
        dev_nr = $file->f_dentry->d_inode->i_sb->s_dev
        inode_nr = $file->f_dentry->d_inode->i_ino
    }
    if (dev_nr == ($1 << 20 | $2) # major/minor device
     && inode_nr == $3)
        printf ("%s(%d) %s 0x%x/%u\n", execname(), pid(), ppfunc(), dev_nr, inode_nr)
}

# stat -c "%D %i" /etc/crontab
fd03 133099
# stap inode-watch.stp 0xfd 3 133099
more(30789) vfs_read 0xfd00003/133099
more(30789) vfs_read 0xfd00003/133099
```

## 3.3 Functions

> Functions are conveniently packaged reusable software: it would be a shame to have to duplicate a complex condition expression or logging directive in every placed it’s used. So, systemtap lets you define functions of your own. Like global variables, systemtap functions may be defined anywhere in the script. They may take any number of string or numeric arguments (by value), and may return a single string or number. The parameter types are inferred as for ordinary variables, and must be consistent throughout the program.
Local and global script variables are available, but target variables are not. That’s because there is no specific debugging-level context associated with a function.

函数是打包好的可重复使用的软件，可惜的是：必须在每个使用的位置重复复制一个复杂的条件表达式或日志记录指令。因此，Systemtap允许定义自己的函数。像全局变量一样，可以在脚本中的任何位置定义Systemtap函数。它们接受任意数量的字符串或数字参数（按值），并返回单个字符串或数字。参数类型的推导与普通变量相同，并且在整个程序中必须保持一致。本地和全局脚本变量可用，但目标变量不可用。这是因为没有与函数关联的特定调试级上下文（译注：即定义函数时没有与某个内核代码相关联）。

> A function is defined with the keyword function followed by a name. Then comes a comma-separated formal argument list (just a list of variable names). The { }-enclosed body consists of any list of statements, including expressions that call functions. Recursion is possible, up to a nesting depth limit. Figure 6 displays function syntax.

函数由关键字**function**定义，后跟函数名。然后是一个逗号分隔的形参列表（只是一个变量名列表）。大括号内的函数体由任何语句列表组成，包括调用函数的表达式。可以递归，直到嵌套深度限制。下面的脚本展示来函数语法：

```sh
# Red Hat convention; see /etc/login.defs UID_MIN
function system_uid_p (u) { return u < 500 }
# kernel device number assembly macro
function makedev (major,minor) { return major << 20 | minor }
function trace_common ()
{
    printf("%d %s(%d)", gettimeofday_s(), execname(), pid())
    # no return value necessary
}
function fibonacci (i)
{
    if (i < 1) return 0
    else if (i < 2) return 1
    else return fibonacci(i-1) + fibonacci(i-2)
}
```

## 3.4 Arrays

> Often, probes will want to share data that cannot be represented as a simple scalar value. Much data is
naturally tabular in nature, indexed by some tuple of thread numbers, processor ids, names, time, and so
on. Systemtap offers associative arrays for this purpose. These arrays are implemented as hash tables with
a maximum size that is fixed at startup. Because they are too large to be created dynamically for individual
probes handler runs, they must be declared as global.

通常，探针将希望共享无法表示为简单标量值的数据。很多数据是本质上是表格形式的，由一些线程号，处理器ID，名称，时间等元组索引。Systemtap为此提供了关联数组。这些数组被实现为哈希表，在启动时确定大小。因为它们太大而无法在每个探针handler运行时动态创建，必须将它们声明为全局。

```sh
global a         declare global scalar or array variable
global b[400]    declare array, reserving space for up to 400 tuples
```

> The basic operations for arrays are setting and looking up elements. These are expressed in awk syntax: the
array name followed by an opening [ bracket, a comma-separated list of index expressions, and a closing ]
bracket. Each index expression may be string or numeric, as long as it is consistently typed throughout the
script.

数组的基本操作是设置和查找元素。这些以awk语法表示：数组名称、后跟半个中括号\[、索引表达式的逗号分隔列表、结尾的半个中括号\]。每个索引表达式可以是字符串或数字，只要在整个脚本中保持一致的类型。

```sh
foo [4,"hello"] ++                      increment the named array slot
processusage [uid(),execname()] ++      update a statistic
times [tid()] = get_cycles()            set a timestamp reference point
delta = get_cycles() - times [tid()]    compute a timestamp delta
```

> Array elements that have not been set may be fetched, and return a dummy null value (zero or an empty
string) as appropriate. However, assigning a null value does not delete the element: an explicit delete
statement is required. Systemtap provides syntactic sugar for these operations, in the form of explicit
membership testing and deletion.

可以获取未设置的数组元素，并返回一个空值（零或空字符串）。但是，分配空值不会删除一个元素：必须使用显示detele语句。Systemtap提供数组成员测试和删除的语法糖。

```sh
if ([4,"hello"] in foo) { }         membership test
delete times[tid()]                 deletion of a single element
delete times                        deletion of all elements
```

> One final and important operation is iteration over arrays. This uses the keyword foreach. Like awk, this
creates a loop that iterates over key tuples of an array, not just values. In addition, the iteration may be sorted
by any single key or the value by adding an extra + or - code.

最后一个重要操作是使用关键字**foreach**在数组上进行迭代。像awk一样创建一个循环，该循环遍历数组的键元组，而不仅仅是值。另外，可以通过在任何单个键或值添加额外的 + 或 - 来对迭代进行排序。

> The break and continue statements work inside foreach loops, too. Since arrays can be large but probe
handlers must not run for long, it is a good idea to exit iteration early if possible. The limit option in the
foreach expression is one way. For simplicity, systemtap forbids any modification of an array while it is being
iterated using a foreach.

break和continue也可以在foreach循环中使用。由于数组可能很大，但探测handler不能运行很长时间，如果可能的话，最好尽早退出迭代，可以通过**limit**来控制。为简单起见，使用foreach进行迭代的时候禁止对数组进行任何修改。

```sh
foreach (x = [a,b] in foo) { fuss_with(x) }     simple loop in arbitrary sequence
foreach ([a,b] in foo+ limit 5) { }             loop in increasing sequence of value, stop
                                                after 5
foreach ([a-,b] in foo) { }                     loop in decreasing sequence of first key
```

## 3.5 Aggregates

> When we said above that values can only be strings or numbers, we lied a little. There is a third type: statistics aggregates, or aggregates for short. Instances of this type are used to collect statistics on numerical values, where it is important to accumulate new data quickly (without exclusive locks) and in large volume (storing only aggregated stream statistics). This type only makes sense for global variables, and may be stored individually or as elements of an array.

当我们在上面说值只能是字符串或数字时，我们撒谎了。有第三种类型：**statistics aggregates**或简称**aggregates**。此类型的实例用于收集有关数值的统计信息，在此情况下，快速（无排他锁）和大量存储新数据（仅存储聚合流统计信息）非常重要。此类型仅对全局变量有意义，并且可以单独存储或作为数组元素存储。

> To add a value to a statistics aggregate, systemtap uses the special operator <<<. Think of it like C++’s << output streamer: the left hand side object accumulates the data sample given on the right hand side. This operation is efficient (taking a shared lock) because the aggregate values are kept separately on each processor, and are only aggregated across processors on request.

要将值添加到aggregates中，Systemtap使用特殊操作符 **<<<**。可以将其想象为C++的 **<<** 输出流送器：左侧对象累积右侧给出的数据样本。此操作符效率很高（采用共享锁），因为聚合值分别保存在每个处理器上，并且仅应请求在各个处理器之间聚合。

```sh
a <<< delta_timestamp
writes[execname()] <<< count
```

> To read the aggregate value, special functions are available to extract a selected statistical function. The aggregate value cannot be read by simply naming it as if it were an ordinary variable. These operations take an exclusive lock on the respective globals, and should therefore be relatively rare. The simple ones are: @min, @max, @count, @avg, and @sum, and evaluate to a single number. In addition, histograms of the data stream may be extracted using the @hist_log and @hist_linear. These evaluate to a special sort of array that may at present only be printed.

要读取aggregate值，可以使用特殊函数来提取选定的统计函数。**不能通过简单地将其命名为普通变量来读取聚合值**。这些操作在各自的全局变量上具有排他锁，因此应该比较少见。简单的是：@min，@max，@count，@avg和@sum，并求值为单个数字。另外，可以使用@hist_log和@hist_linear提取数据流的直方图。这些值评估为一种可打印的特殊数组。

```sh
@avg(a)                             the average of all the values accumulated into a.
print(@hist_linear(a,0,100,10))     print an “ascii art” linear histogram of the same data
                                    stream a, bounds 0 . . . 100, bucket width is 10
@count(writes["zsh"])               the number of times “zsh” ran the probe handler.
print(@hist_log(writes["zsh"]))     print an “ascii art” logarithmic histogram of the same
                                    data stream writes.
```

## 3.6 Safety

> The full expressivity of the scripting language raises good questions of safety. Here is a set of Q&A:

脚本语言的完整表达能力产生了一些安全性问题。下面是一组问答：

> What about infinite loops? recursion? A probe handler is bounded in time. The C code generated by systemtap includes explicit checks that limit the total number of statements executed to a small number. A similar limit is imposed on the nesting depth of function calls. When either limit is exceeded, that probe handler cleanly aborts and signals an error. The systemtap session is normally configured to abort as a whole at that time.

What about infinite loops? recursion? 探针handler受时间限制。由Systemtap脚本生成的C代码包括显式检查，这些检查将**执行的语句总数**限制为少量。
对**函数调用的嵌套深度**也施加了类似的限制。当超过任一限制时，探针handler彻底中止并发出错误信号。Systemtap会话通常配置整体中止。

> What about running out of memory? No dynamic memory allocation whatsoever takes place during the
execution of probe handlers. Arrays, function contexts, and buffers are allocated during initialization.
These resources may run out during a session, and generally result in errors.

What about running out of memory? 在探针handler期间，不会进行任何动态内存分配。数组，函数上下文和缓冲区全在初始化过程中分配。这些资源可能在会话期间用完，用完后会抛出错误。

> What about locking? If multiple probes seek conflicting locks on the same global variables, one or more
of them will time out, and be aborted. Such events are tallied as “skipped” probes, and a count is
displayed at session end. A configurable number of skipped probes can trigger an abort of the session.

What about locking? 如果多个探针在相同的全局变量上获取同一把锁，则它们中的一个或多个将超时，并且被中止。将此类event记为“skipped”探针，并且在会话结束时会显示一个计数。可配置skipped探针的数量来终止会话。

> What about null pointers? division by zero? The C code generated by systemtap translates potentially
dangerous operations to routines that check their arguments at run time. These signal errors if they are
invalid. Many arithmetic and string operations silently overflow if the results exceed representation
limits.

What about null pointers? division by zero? Systemtap在翻译危险的操作时，可能会在运行时检查其例程的参数。如果无效，会发出信号来表明错误。如果结果超出表示范围，许多算术和字符串运算都会静默溢出。

> What about bugs in the translator? compiler? While bugs in the translator, or the runtime layer certainly
exist4, our test suite gives some assurance. Plus, the entire generated C code may be inspected (try the
-p3 option). Compiler bugs are unlikely to be of any greater concern for systemtap than for the kernel
as a whole. In other words, if it was reliable enough to build the kernel, it will build the systemtap
modules properly too.

What about bugs in the translator? compiler? 虽然翻译器或运行时层中确实存在错误的exist（参考[bugzilla](http://sourceware.org/bugzilla)），我们的测试套件会提供一些保障。另外，可以检查整个生成的C代码（尝试-p3选项）。与内核相比，systemtap不太可能关心编译器错误。换句话说，如果足以可靠地构建内核，它将构建systemtap模块也正确。

> Is that the whole truth? In practice, there are several weak points in systemtap and the underlying
kprobes system at the time of writing. Putting probes indiscriminately into unusually sensitive parts
of the kernel (low level context switching, interrupt dispatching) has reportedly caused crashes in the
past. We are fixing these bugs as they are found, and constructing a probe point “blacklist”, but it is
not complete.

Is that the whole truth? 在实践中，Systemtap及kprobes系统存在一些弱点。在过去，随意将探针放入异常敏感（进程上下文切换，中断调度）的内核部分会造成内核crash。我们正在修复发现的这些错误，并构建了一个探测点“黑名单”，但不完整。

## 3.7 Exercises

略。

# 4 Tapsets

> After writing enough analysis scripts for yourself, you may become known as an expert to your colleagues, who will want to use your scripts. Systemtap makes it possible to share in a controlled manner; to build libraries of scripts that build on each other. In fact, all of the functions (pid(), etc.) used in the scripts above come from tapset scripts like that. A “tapset” is just a script that designed for reuse by installation into a special directory.

在为自己编写了足够的分析脚本之后，你可能会成为同事眼中的专家，他们将希望使用你的脚本。Systemtap以受控制的方式使共享成为可能；建立相互的脚本库。实际上，以上脚本中使用的所有函数（`pid()`等）都来自类似的tapset脚本。一个“tapset”只是一个脚本，旨在通过安装到特殊目录中来重复使用。

## 4.1 Automatic selection

> Systemtap attempts to resolve references to global symbols (probes, functions, variables) that are not defined within the script by a systematic search through the tapset library for scripts that define those symbols. Tapset scripts are installed under the default directory named /usr/share/systemtap/tapset. A user may give additional directories with the -I DIR option. Systemtap searches these directories for script (.stp) files.

Systemtap尝试通过在Tapset库中**系统搜索**定义这些符号的脚本来尝试解析对全局符号（探针，函数，变量）的引用。Tapset脚本安装在名为/usr/share/systemtap/tapset的默认目录下。用户可以使用-I DIR选项指定其他目录。Systemtap在这些目录中搜索脚本（.stp）文件。

> The search process includes subdirectories that are specialized for a particular kernel version and/or architecture, and ones that name only larger kernel families. Naturally, the search is ordered from specific to general, as shown in Figure 7. 

搜索过程包括专门用于特定内核版本和/或体系结构的子目录，以及仅命名较大内核家族的子目录。一般来说，搜索是从特定顺序到一般顺序的，如下所示：

```sh
# stap -p1 -vv -e ’probe begin { }’ > /dev/null
Created temporary directory "/tmp/staplnEBh7"
Searched ’/usr/share/systemtap/tapset/2.6.15/i686/*.stp’, match count 0
Searched ’/usr/share/systemtap/tapset/2.6.15/*.stp’, match count 0
Searched ’/usr/share/systemtap/tapset/2.6/i686/*.stp’, match count 0
Searched ’/usr/share/systemtap/tapset/2.6/*.stp’, match count 0
Searched ’/usr/share/systemtap/tapset/i686/*.stp’, match count 1
Searched ’/usr/share/systemtap/tapset/*.stp’, match count 12
Pass 1: parsed user script and 13 library script(s) in 350usr/10sys/375real ms.
Running rm -rf /tmp/staplnEBh7
```

> When a script file is found that defines one of the undefined symbols, that entire file is added to the probing session being analyzed. This search is repeated until no more references can become satisfied. Systemtap signals an error if any are still unresolved.

当找到定义了未定义符号之一的脚本文件时，会将整个文件添加到正在分析的探测会话中。重复此搜索，直到所有引用全被解析。最后，如果由为解析的引用，Systemtap会发出信号以示错误。

> This mechanism enables several programming idioms. First, it allows some global symbols to be defined only for applicable kernel version/architecture pairs, and cause an error if their use is attempted on an inapplicable host. Similarly, the same symbol can be defined differently depending on kernels, in much the same way that different kernel include/asm/ARCH/ files contain macros that provide a porting layer. 

这种机制使用了几种编程习惯用法：首先，它只允许为适用的内核版本/体系结构对定义全局符号，如果在不适用的主机上尝试使用它们，则会导致错误。类似地，可以根据内核不同地定义相同的符号，其方式与内核中include/asm/ARCH/文件中提供的移植层的宏相似。

> Another use is to separate the default parameters of a tapset routine from its implementation. For example, consider a tapset that defines code for relating elapsed time intervals to process scheduling activities. The data collection code can be generic with respect to which time unit (jiffies, wall-clock seconds, cycle counts) it can use. It should have a default, but should not require additional run-time checks to let a user choose another. Figure 8 shows a way.

另一个用途是将tapset例程的默认参数与其实现分开。例如，考虑一个tapset，它定义了将经过的时间间隔与调度活动相关联的代码。数据收集代码可以使用哪个时间单位（抖动，挂钟秒数，周期计数）通用。它应该具有默认值，但不应要求其他运行时检查来允许用户选择其他选项。如下：

```sh
# cat tapset/time-common.stp
global __time_vars
function timer_begin (name) { __time_vars[name] = __time_value () }
function timer_end (name) { return __time_value() - __time_vars[name] }

# cat tapset/time-default.stp
function __time_value () { return gettimeofday_us () }

# cat tapset-time-user.stp
probe begin
{
    timer_begin ("bench")
    for (i=0; i<100; i++) ;
        printf ("%d cycles\n", timer_end ("bench"))
    exit ()
}
function __time_value () { return get_ticks () } # override for greater precision
```

> A tapset that exports only data may be as useful as ones that exports functions or probe point aliases (see below). Such global data can be computed and kept up-to-date using probes internal to the tapset. Any outside reference to the global variable would incidentally activate all the required probes.

仅导出数据的tapset可能与导出函数或探测点别名的tapset一样有用（请参见下文）。可以使用tapset内部的探针来计算并保持此类全局数据为最新。任何对全局变量的外部引用都将偶然激活所有必需的探针。

## 4.2 Probe point aliases

> Probe point aliases allow creation of new probe points from existing ones. This is useful if the new probe points are named to provide a higher level of abstraction. For example, the system-calls tapset defines probe point aliases of the form syscall.open etc., in terms of lower level ones like kernel.function("sys_open"). Even if some future kernel renames sys_open, the aliased name can remain valid.

探针点别名允许从现有的探针点创建新的探针点。如果新的探针点被命名为提供更高级别的抽象，这将很有用。例如，系统调用tapset定义了`syscall.open`等形式的探测点别名来代替`kernel.function（“sys_open”）`之类的较低级别别名。即使将来的某些内核将sys_open重命名来，别名也可以保持有效。

> A probe point alias definition looks like a normal probe. Both start with the keyword probe and have a probe handler statement block at the end. But where a normal probe just lists its probe points, an alias creates a new name using the assignment (=) operator. Another probe that names the new probe point will create an actual probe, with the handler of the alias prepended.

探针点别名定义看起来像普通探针。两者都以关键字probe开头，并在结尾处有probe handler语句块。但是在普通探针仅列出其探针点的地方，别名使用赋值（=）运算符创建了一个新名称。命名新探测点的另一个探测将创建一个实际的探测，并使用别名的处理程序作为前缀。

> This prepending behavior serves several purposes. It allows the alias definition to “preprocess” the context of the probe before passing control to the user-specified handler. This has several possible uses:

这种前置行为有几个目的。它允许别名在将控制权传递给用户指定的处理程序之前，对探针的上下文进行“预处理”。这有几种可能的用途：

```sh
if ($flag1 != $flag2)       next skip probe unless given condition is met
name = "foo"                supply probe-describing values
var = $var                  extract target variable to plain local variable
```

> Figure 9 demonstrates a probe point alias definition as well as its use. It demonstrates how a single probe point alias can expand to multiple probe points, even to other aliases. It also includes probe point wildcarding. These functions are designed to compose sensibly.

下面的脚本演示了探针别名的定义及其用法。它演示了单个探测点别名如何扩展到多个探测点，甚至扩展到其他别名。它还包括探测点通配符。这些功能旨在合理组合。 

```sh
# cat probe-alias.stp
probe syscallgroup.io = syscall.open, syscall.close, syscall.read, syscall.write { groupname = "io" }
probe syscallgroup.process = syscall.fork, syscall.execve { groupname = "process" }
probe syscallgroup.* { groups [execname() . "/" . groupname] ++ }
probe end
{
    foreach (eg+ in groups)
        printf ("%s: %d\n", eg, groups[eg])
}
global groups

# stap probe-alias.stp
05-wait_for_sys/io: 19
10-udev.hotplug/io: 17
20-hal.hotplug/io: 12
X/io: 73
apcsmart/io: 59
[...]
make/io: 515
make/process: 16
[...]
xfce-mcs-manage/io: 3
xfdesktop/io: 5
[...]
xmms/io: 7070
zsh/io: 78
zsh/process: 5
```

## 4.3 Embedded C

> Sometimes, a tapset needs provide data values from the kernel that cannot be extracted using ordinary target variables ($var). This may be because the values are in complicated data structures, may require lock awareness, or are defined by layers of macros. Systemtap provides an “escape hatch” to go beyond what the language can safely offer. In certain contexts, you may embed plain raw C in tapsets, exchanging power for the safety guarantees listed in section 3.6. End-user scripts may not include embedded C code, unless systemtap is run with the -g (“guru” mode) option. Tapset scripts get guru mode privileges automatically.

有时，tapset需要提供无法使用普通目标变量（$var）提取的内核数据值。这可能是因为这些值位于复杂的数据结构中，可能需要锁，或者由宏层定义。Systemtap提供了一个“逃生舱口”，以越过该语言可以安全提供的范围。在某些情况下，你可以将普通的原始C嵌入到tapset中，以3.6节中列出的安全保证为牺牲。最终用户脚本可能不包含嵌入式C代码，除非使用-g（“guru”模式）选项运行Systemtap。 Tapset脚本会自动获得guru模式权限。

> Embedded C can be the body of a script function. Instead enclosing the function body statements in { and }, use %{ and %}. Any enclosed C code is literally transcribed into the kernel module: it is up to you to make it safe and correct. In order to take parameters and return a value, macros STAP_ARG_* and STAP_RETVALUE are made available. The familiar data-gathering functions pid(), execname(), and their neighbours are all embedded C functions. Figure 10 contains another example.

嵌入式C可以是脚本函数的主体。而不是将函数主体语句包含在大括号中，请使用 **％{** 和 **％}**。 任何包含在内的C代码实际上都会直接按照字面量传送到内核模块中：由自己决定使其安全和正确性。为了获取参数并返回值，使宏**STAP_ARG_\***和**STAP_RETVALUE**。熟悉的数据收集函数pid（），execname（）及其系列都嵌入进C函数。如下：

```sh
# cat embedded-C.stp
%{
    #include <linux/sched.h>
    #include <linux/list.h>
%}
function task_execname_by_pid:string (pid:long) %{
    struct task_struct *p;
    struct list_head *_p, *_n;
    list_for_each_safe(_p, _n, &current->tasks) {
        p = list_entry(_p, struct task_struct, tasks);
        if (p->pid == (int)STAP_ARG_pid)
            snprintf(STAP_RETVALUE, MAXSTRINGLEN, "%s", p->comm);
    }
%}

probe begin
{
    printf("%s(%d)\n", task_execname_by_pid(target()), target())
    exit()
}
# pgrep emacs
16641
# stap -g embedded-C.stp -x 16641
emacs(16641)
```

> Since systemtap cannot examine the C code to infer these types, an optional annotation syntax is available to assist the type inference process. Simply suffix parameter names and/or the function name with :string or :long to designate the string or numeric type. In addition, the script may include a %{ %} block at the outermost level of the script, in order to transcribe declarative code like #include <linux/foo.h>.

由于Systemtap无法检查C代码来推断这些类型，因此可以使用optional注释语法来辅助类型推断过程。只需在参数名称和/或函数名称后加上 **:string** 或 **:long** 后缀即可指定字符串或数字类型。另外，可以在脚本的最外层包含％{％}块，以便转录诸如#include <linux/foo.h>之类的声明性代码。

> These enable the embedded C functions to refer to general kernel types. There are a number of safety-related constraints that should be observed by developers of embedded C code.

这些嵌入式C函数能够引用常规内核类型。此类开发人员应遵守许多与安全相关的约束。

- Do not dereference pointers that are not known or testable valid.
- Do not call any kernel routine that may cause a sleep or fault.
- Consider possible undesirable recursion, where your embedded C function calls a routine that may be the subject of a probe. If that probe handler calls your embedded C function, you may suffer infinite regress. Similar problems may arise with respect to non-reentrant locks.
- If locking of a data structure is necessary, use a **trylock** type call to attempt to take the lock. If that
fails, give up, do not block.

## 4.4 Naming conventions

> Using the tapset search mechanism just described, potentially many script files can become selected for inclusion in a single session. This raises the problem of name collisions, where different tapsets accidentally use the same names for functions/globals. This can result in errors at translate or run time.

使用刚刚描述的tapset搜索机制，可能会在单个会话中选择许多脚本文件。这会引起了名称冲突的问题，其中不同的Tapset意外地将相同的名称用于函数/全局变量。这可能会导致翻译或运行时错误。

> To control this problem, systemtap tapset developers are advised to follow naming conventions. Here is
some of the guidance.

为了避免类似问题，Systemtap tapset开发人员可以采取以下建议：

- Pick a unique name for your tapset, and substitute it for TAPSET below.
- Separate identifiers meant to be used by tapset users from those that are internal implementation artifacts.
- Document the first set in the appropriate man pages.
- Prefix the names of external identifiers with TAPSET if there is any likelihood of collision with other tapsets or end-user scripts.
- Prefix any probe point aliases with an appropriate prefix.
- Prefix the names of internal identifiers with _TAPSET_.

## 4.5 Exercises

略。

# 5 Further information

下面是一些手册：

- stap： systemtap program usage, language summary
- stappaths： your systemtap installation paths
- stapprobes： probes / probe aliases provided by built-in tapsets
- stapex： a few basic example scripts
- tapset::*： summaries of the probes and functions in each tapset
- probe::*： detailed descriptions of each probe
- function::*： detailed descriptions of each function

There is much more documentation and sample scripts included. You may find them under /usr/share/doc/systemtap*/.

> Then, there is the source code itself. Since systemtap is free software, you should have available the entire
source code. The source files in the tapset/ directory are also packaged along with the systemtap binary.
Since systemtap reads these files rather than their documentation, they are the most reliable way to see
what’s inside all the tapsets. Use the -v (verbose) command line option, several times if you like, to show
inner workings.

然后，有源代码本身可以参考。由于Systemtap是免费软件，因此你应该拥有完整的源代码。tapset目录中的源文件也与Systemtap二进制文件一起打包。由于Systemtap读取这些文件而不是其文档，因此查看tapset里面的源码是最可靠方式。使用-v（详细）命令行选项，如果需要，可以多次显示内部运作。

Finally, there is the project web site (http://sourceware.org/systemtap/) with several articles, an archived
public mailing list for users and developers (systemtap@sourceware.org), IRC channels, and a live GIT
source repository. Come join us!