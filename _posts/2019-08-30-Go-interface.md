---
layout: post
title: 'Go Interface'
subtitle: '未整理'
date: 2019-08-30
categories: 技术
cover: ''
tags: Golang
---

环境：
```go
$ go version
go version go1.12.7 linux/amd64
$ uname -a
18.04.1-Ubuntu SMP x86_64 x86_64 x86_64 GNU/Linux
```

```go
package main

type Mather interface {
	Add(a, b int32) int32
	Sub(a, b int64) int64
}

type Adder struct {
	id  int32
	str string
}

//go:noinline
func (adder Adder) Add(a, b int32) int32 { return a + b }

//go:noinline
func (adder Adder) Sub(a, b int64) int64 { return a - b }

//go:noinline
//func (adder Adder) Test(a, b int64) int64 { return a - b }
// 编译器会生成Test()的wrapper "".(*Adder).Test，但是不会在go.itab里面给它留位置

func main() {
	m := Mather(Adder{id: 6754, str: "test"})

	// This call just makes sure that the interface is actually used.
	// Without this call, the linker would see that the interface defined above
	// is in fact never used, and thus would optimize it out of the final
	// executable.
	m.Add(10, 32)
}
```
```go
"".main STEXT size=159 args=0x0 locals=0x40
	0x0000 00000 (main.go:19)	TEXT	"".main(SB), ABIInternal, $64-0
	0x0000 00000 (main.go:19)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:19)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:19)	JLS	149
	0x0013 00019 (main.go:19)	SUBQ	$64, SP
	0x0017 00023 (main.go:19)	MOVQ	BP, 56(SP)
	0x001c 00028 (main.go:19)	LEAQ	56(SP), BP
	0x0021 00033 (main.go:20)	MOVL	$0, ""..autotmp_1+32(SP)
	0x0029 00041 (main.go:20)	XORPS	X0, X0
	0x002c 00044 (main.go:20)	MOVUPS	X0, ""..autotmp_1+40(SP)
	0x0031 00049 (main.go:20)	MOVL	$6754, ""..autotmp_1+32(SP)
	0x0039 00057 (main.go:20)	LEAQ	go.string."test"(SB), AX
	0x0040 00064 (main.go:20)	MOVQ	AX, ""..autotmp_1+40(SP)
	0x0045 00069 (main.go:20)	MOVQ	$4, ""..autotmp_1+48(SP)
	0x004e 00078 (main.go:20)	LEAQ	go.itab."".Adder,"".Mather(SB), AX
	0x0055 00085 (main.go:20)	MOVQ	AX, (SP)
	0x0059 00089 (main.go:20)	LEAQ	""..autotmp_1+32(SP), AX
	0x005e 00094 (main.go:20)	MOVQ	AX, 8(SP)
	0x0063 00099 (main.go:20)	CALL	runtime.convT2I(SB)
	0x0068 00104 (main.go:20)	MOVQ	16(SP), AX               # iface.itab
	0x006d 00109 (main.go:20)	MOVQ	24(SP), CX               # iface.data
	0x0072 00114 (main.go:26)	MOVQ	24(AX), AX               # 这里决定了调用itab.fun[0]
	0x0076 00118 (main.go:26)	MOVQ	CX, (SP)
	0x007a 00122 (main.go:26)	MOVQ	$137438953482, CX
	0x0084 00132 (main.go:26)	MOVQ	CX, 8(SP)
	0x0089 00137 (main.go:26)	CALL	AX
	0x008b 00139 (main.go:27)	MOVQ	56(SP), BP
	0x0090 00144 (main.go:27)	ADDQ	$64, SP
  0x0094 00148 (main.go:27)	RET
```
```go
func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
  ...
  0x100807f		4883ec38		SUBQ $0x38, SP		
  0x1008083		48896c2430		MOVQ BP, 0x30(SP)	
  0x1008088		488d6c2430		LEAQ 0x30(SP), BP	
	t := tab._type
  0x100808d		488b442440		MOVQ 0x40(SP), AX	
  0x1008092		488b4808		MOVQ 0x8(AX), CX	
  0x1008096		48894c2428		MOVQ CX, 0x28(SP)	
	x := mallocgc(t.size, t, true)
  0x100809b		488b11			MOVQ 0(CX), DX			
  0x100809e		48891424		MOVQ DX, 0(SP)			
  0x10080a2		48894c2408		MOVQ CX, 0x8(SP)		
  0x10080a7		c644241001		MOVB $0x1, 0x10(SP)		
  0x10080ac		e8af190000		CALL runtime.mallocgc(SB)	
  0x10080b1		488b442418		MOVQ 0x18(SP), AX		
  0x10080b6		4889442420		MOVQ AX, 0x20(SP)		
	typedmemmove(t, x, elem)
  0x10080bb		488b4c2428		MOVQ 0x28(SP), CX		
  0x10080c0		48890c24		MOVQ CX, 0(SP)			
  0x10080c4		4889442408		MOVQ AX, 0x8(SP)		
  0x10080c9		488b4c2448		MOVQ 0x48(SP), CX		
  0x10080ce		48894c2410		MOVQ CX, 0x10(SP)		
  0x10080d3		e808530000		CALL runtime.typedmemmove(SB)	
	return
  0x10080d8		488b442440		MOVQ 0x40(SP), AX	
  0x10080dd		4889442450		MOVQ AX, 0x50(SP)	
  0x10080e2		488b442420		MOVQ 0x20(SP), AX	
  0x10080e7		4889442458		MOVQ AX, 0x58(SP)	
  0x10080ec		488b6c2430		MOVQ 0x30(SP), BP	
  0x10080f1		4883c438		ADDQ $0x38, SP		
  0x10080f5		c3			RET			
```
```go
    +---------+
    |         |
    |         |
    +---------+
    |   rbp   |
    +---------+
    |         |
    +---------+
    |   *  |----------------> "test"
    +---------+
+-> |  6754   |
|   +---------+ ++
|   |    x    |  | -> iface.data
|   +---------+  | make in convT2I
|   |  itab   |  | -> iface.itab
|   +---------+ ++
+-+ |  elem   |
    +---------+
    |  itab   |
    +---------+ <+ rsp
    |  ret    |
    +---------+
    |  rbp    |
    +---------+
    |    t    |
    +---------+
    |    x    |
    +---------+
    |         |
    +---------+
    |         |
    +---------+
    |         |
    +---------+
    |         |
    +---------+ <+ rsp
```
After `CALL	runtime.convT2I(SB)`:
```go
    +---------+
    |         |
    |         |
    +---------+
    |   rbp   |
    +---------+
    |         |
    +---------+
    |   *  |----------------> "test"
    +---------+
    |  6754   |
    +---------+ ++
    |    x    |  | 
    +---------+  | make in convT2I
    |  itab   |  |
    +---------+ ++
    |  10;32  |
    +---------+
    |   x     | receiver;iface.data, point to the Adder in the heap
    +---------+ <+ rsp
```
这里和call method流程一样了，We move x to the top of stack, as argument #1, to satisfy the calling convention: the receiver for a method should always be passed as the first argument.

`CALL	AX` 等价于 `CALL main.(*Adder).Add()`, wrapper method for `main.Adder.Add()`

receiver 是 ptr, *Adder 也是 ptr, 满足要求

参考资料：
- [Interfaces and other types](https://golang.org/doc/effective_go.html#interfaces_and_types)
- [Understanding Go Interfaces](https://www.youtube.com/watch?v=F4wUrj6pmSI&feature=youtu.be&t=285)
- [Go Data Structures: Interfaces](https://research.swtch.com/interfaces)
- [Chapter II: Interfaces](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/README.md)