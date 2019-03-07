---
layout: post
title: "Linux Programmer's Manual"
comments: true
description: ""
keywords: "Markdown, Linux, Manual, Computer, Program"
---

```c
pid_t waitpid(pid_t pid, int *status, int options);
```

>All of these system calls are used to wait for **state changes** in a child of the calling process, and obtain information about the child whose state **has changed**. A state change is considered to be: the child **terminated**; the child was **stopped** by a signal; or the child was **resumed** by a signal. If a  child has already changed state, then these calls return immediately. Otherwise they **block** until either a child changes state or a signal handler interrupts the call.