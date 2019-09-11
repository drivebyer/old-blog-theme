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

type Adder1 struct {
	id int32
	srt string 
}

//go:noinline
func (adder1 Adder1) Add(a, b int32) int32 { return a + b }

func main() {
	m := Mather(Adder{id: 6754, str: "test"})
	//m := Mather(Adder1{id: 6754, str: "test"})
	// This call just makes sure that the interface is actually used.
	// Without this call, the linker would see that the interface defined above
	// is in fact never used, and thus would optimize it out of the final
	// executable.
  ret := m.Add(10, 32)
  fmt.Println(ret) // ignore, just for explain
}
```
```go
// ignore GC related and stack overflow check.
"".main STEXT size=258 args=0x0 locals=0x70
	0x0000 00000 (main.go:25)	TEXT	"".main(SB), ABIInternal, $112-0
	0x0000 00000 (main.go:25)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:25)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:25)	JLS	248
	0x0013 00019 (main.go:25)	SUBQ	$112, SP
	0x0017 00023 (main.go:25)	MOVQ	BP, 104(SP)
	0x001c 00028 (main.go:25)	LEAQ	104(SP), BP
	0x0021 00033 (main.go:26)	MOVL	$0, ""..autotmp_6+80(SP)
	0x0029 00041 (main.go:26)	XORPS	X0, X0
	0x002c 00044 (main.go:26)	MOVUPS	X0, ""..autotmp_6+88(SP)
	0x0031 00049 (main.go:26)	MOVL	$6754, ""..autotmp_6+80(SP)
	0x0039 00057 (main.go:26)	LEAQ	go.string."test"(SB), AX
	0x0040 00064 (main.go:26)	MOVQ	AX, ""..autotmp_6+88(SP)
	0x0045 00069 (main.go:26)	MOVQ	$4, ""..autotmp_6+96(SP)
	0x004e 00078 (main.go:26)	LEAQ	go.itab."".Adder,"".Mather(SB), AX
	0x0055 00085 (main.go:26)	MOVQ	AX, (SP)
	0x0059 00089 (main.go:26)	LEAQ	""..autotmp_6+80(SP), AX
	0x005e 00094 (main.go:26)	MOVQ	AX, 8(SP)
	0x0063 00099 (main.go:26)	CALL	runtime.convT2I(SB)
	0x0068 00104 (main.go:26)	MOVQ	24(SP), AX
	0x006d 00109 (main.go:26)	MOVQ	16(SP), CX
	0x0072 00114 (main.go:32)	MOVQ	24(CX), CX
	0x0076 00118 (main.go:32)	MOVQ	AX, (SP)
	0x007a 00122 (main.go:32)	MOVQ	$137438953482, AX
	0x0084 00132 (main.go:32)	MOVQ	AX, 8(SP)
	0x0089 00137 (main.go:32)	CALL	CX
	0x008b 00139 (main.go:32)	MOVL	16(SP), AX
	0x008f 00143 (main.go:33)	MOVL	AX, (SP)
	0x0092 00146 (main.go:33)	CALL	runtime.convT32(SB)
```
```go
go.itab."".Adder,"".Mather SRODATA dupok size=40
	0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
	0x0010 8a 3d 5f 61 00 00 00 00 00 00 00 00 00 00 00 00  .=_a............
	0x0020 00 00 00 00 00 00 00 00                          ........
	rel 0+8 t=1 type."".Mather+0
	rel 8+8 t=1 type."".Adder+0
	rel 24+8 t=1 "".(*Adder).Add+0
	rel 32+8 t=1 "".(*Adder).Sub+0
```
在调用`CALL	runtime.convT2I(SB)`前，栈桢如下：
```go
      +-----------+
      |           |
      +-----------+
      |    4      |
      +-----------+
      |           |
      +-----------+
      |           |
      +-----------+
      |     *   |------------>  "test"
      +-----------+
+---> |   6754    |
|     +-----------+
|     |           |
|     +-----------+
|     |           |
|     +-----------+
|     |           |
|     +-----------+
|     |           |
|     +-----------+
|     |           |
|     +-----------+
|     |           |
|     +-----------+
+----------|      |
      +-----------+
      |   *itab   |
      +-----------+ <+ rsp
```

```go
func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
  0x1008820		65488b0c2530000000	MOVQ GS:0x30, CX
  0x1008829		483b6110	  	CMPQ 0x10(CX), SP
  0x100882d		7677			    JBE 0x10088a6
  0x100882f		4883ec38		  SUBQ $0x38, SP
  0x1008833		48896c2430		MOVQ BP, 0x30(SP)
  0x1008838		488d6c2430		LEAQ 0x30(SP), BP
	t := tab._type
  0x100883d		488b442440		MOVQ 0x40(SP), AX    ;; 0x40(SP) = *itab
  0x1008842		488b4808		  MOVQ 0x8(AX), CX     ;; itab._type
  0x1008846		48894c2428		MOVQ CX, 0x28(SP)
	x := mallocgc(t.size, t, true)
  0x100884b		488b11			  MOVQ 0(CX), DX
  0x100884e		48891424		  MOVQ DX, 0(SP)
  0x1008852		48894c2408		MOVQ CX, 0x8(SP)
  0x1008857		c644241001		MOVB $0x1, 0x10(SP)
  0x100885c		e81f1b0000		CALL runtime.mallocgc(SB)
  0x1008861		488b442418		MOVQ 0x18(SP), AX
  0x1008866		4889442420		MOVQ AX, 0x20(SP)
	typedmemmove(t, x, elem)
  0x100886b		488b4c2428		MOVQ 0x28(SP), CX
  0x1008870		48890c24		  MOVQ CX, 0(SP)
  0x1008874		4889442408		MOVQ AX, 0x8(SP)
  0x1008879		488b4c2448		MOVQ 0x48(SP), CX
  0x100887e		48894c2410		MOVQ CX, 0x10(SP)
  0x1008883		e8087c0000		CALL runtime.typedmemmove(SB)
	return
  0x1008888		488b442440		MOVQ 0x40(SP), AX
  0x100888d		4889442450		MOVQ AX, 0x50(SP)
  0x1008892		488b442420		MOVQ 0x20(SP), AX
  0x1008897		4889442458		MOVQ AX, 0x58(SP)
  0x100889c		488b6c2430		MOVQ 0x30(SP), BP
  0x10088a1		4883c438		  ADDQ $0x38, SP
  0x10088a5		c3			      RET		
```
在执行`RET`前，函数的栈桢如下：
```go

```

```go
      +-----------+
      |           |
      +-----------+
      |    4      |
      +-----------+
      |           |
      +-----------+
      |           |
      +-----------+
      |     *   |------------>  "test"
      +-----------+
+---> |   6754    |
|     +-----------+
|     |           |
|     +-----------+
|     |           |
|     +-----------+
|     |           |
|     +-----------+
|     |           |
|     +-----------+
|     |     x   |---------> point to heap
|     +-----------+
|     |   *itab   |
|     +-----------+
+----------|      |
      +-----------+
      |   *itab   |
      +-----------+ 
      |    ret    |
      +-----------+ <+ rsp
```

下面继续回到main函数，在执行`CALL	CX`前，栈桢如下：
```go
+-----------+
|           |
+-----------+
|    4      |
+-----------+
|           |
+-----------+
|           |
+-----------+
|     *   |------------>  "test"
+-----------+
|   6754    |
+-----------+
|           |
+-----------+
|           |
+-----------+
|           |
+-----------+
|           |
+-----------+
|     x   |---------> point to heap
+-----------+         ^
|   *itab   |         |
+-----------+         |
|   32 10   |         |
+-----------+         |
|     x   |-----------+
+-----------+ <+ rsp
```

到这里和call method流程一样了，We move x to the top of stack, as argument #1, to satisfy the calling convention: the receiver for a method should always be passed as the first argument.

`CALL	AX` 等价于 `CALL main.(*Adder).Add()`：wrapper method for `main.Adder.Add()`.

receiver 是 ptr, *Adder 也是 ptr, 满足调用要求。

```go
// ignore GC related...
"".(*Adder).Add STEXT dupok size=139 args=0x18 locals=0x30
	0x0000 00000 (<autogenerated>:1)	TEXT	"".(*Adder).Add(SB), DUPOK|WRAPPER|ABIInternal, $48-24
	0x0000 00000 (<autogenerated>:1)	MOVQ	(TLS), CX
	0x0009 00009 (<autogenerated>:1)	CMPQ	SP, 16(CX)
	0x000d 00013 (<autogenerated>:1)	JLS	117
	0x000f 00015 (<autogenerated>:1)	SUBQ	$48, SP
	0x0013 00019 (<autogenerated>:1)	MOVQ	BP, 40(SP)
	0x0018 00024 (<autogenerated>:1)	LEAQ	40(SP), BP
	0x001d 00029 (<autogenerated>:1)	MOVQ	32(CX), BX  ;; BX = g._panic
	0x0021 00033 (<autogenerated>:1)	TESTQ	BX, BX
	0x0024 00036 (<autogenerated>:1)	JNE	124
	0x0026 00038 (<autogenerated>:1)	NOP
	0x0026 00038 (<autogenerated>:1)	MOVQ	""..this+56(SP), AX ;; AX = x
	0x002b 00043 (<autogenerated>:1)	TESTQ	AX, AX 
	0x002e 00046 (<autogenerated>:1)	JEQ	110
	0x0030 00048 (<autogenerated>:1)	MOVQ	8(AX), CX     ;; CX = 10
	0x0034 00052 (<autogenerated>:1)	MOVQ	16(AX), DX    ;; DX = 32
	0x0038 00056 (<autogenerated>:1)	MOVL	(AX), AX      ;; AX = 6754
	0x003a 00058 (<autogenerated>:1)	MOVL	AX, (SP)
	0x003d 00061 (<autogenerated>:1)	MOVQ	CX, 8(SP)
	0x0042 00066 (<autogenerated>:1)	MOVQ	DX, 16(SP)
	0x0047 00071 (<autogenerated>:1)	MOVL	"".a+64(SP), AX ;; AX = x
	0x004b 00075 (<autogenerated>:1)	MOVL	AX, 24(SP)
	0x004f 00079 (<autogenerated>:1)	MOVL	"".b+68(SP), AX
	0x0053 00083 (<autogenerated>:1)	MOVL	AX, 28(SP)
	0x0057 00087 (<autogenerated>:1)	CALL	"".Adder.Add(SB)
	0x005c 00092 (<autogenerated>:1)	MOVL	32(SP), AX
	0x0060 00096 (<autogenerated>:1)	MOVL	AX, "".~r2+72(SP)
	0x0064 00100 (<autogenerated>:1)	MOVQ	40(SP), BP
	0x0069 00105 (<autogenerated>:1)	ADDQ	$48, SP
	0x006d 00109 (<autogenerated>:1)	RET
	0x006e 00110 (<autogenerated>:1)	CALL	runtime.panicwrap(SB)
	0x0073 00115 (<autogenerated>:1)	UNDEF
	0x0075 00117 (<autogenerated>:1)	NOP
	0x0075 00117 (<autogenerated>:1)	CALL	runtime.morestack_noctxt(SB)
	0x007a 00122 (<autogenerated>:1)	JMP	0
	0x007c 00124 (<autogenerated>:1)	LEAQ	56(SP), DI
	0x0081 00129 (<autogenerated>:1)	CMPQ	(BX), DI
	0x0084 00132 (<autogenerated>:1)	JNE	38
	0x0086 00134 (<autogenerated>:1)	MOVQ	SP, (BX)
	0x0089 00137 (<autogenerated>:1)	JMP	38
```
注意第00043行，如果x指针为空（即自动生成的wrapper函数的receiver为空），则执行panicwrap()函数。

在执行`CALL	"".Adder.Add(SB)`前，栈桢如下：
```
+-----------+
|           |
+-----------+
|    4      |
+-----------+
|           |
+-----------+
|           |
+-----------+
|     *   |------------>  "test"
+-----------+
|   6754    |
+-----------+
|           |
+-----------+
|           |
+-----------+
|           |
+-----------+
|           |
+-----------+
|     x   |---------> point to heap
+-----------+         ^
|   *itab   |         |
+-----------+         |
|   32 10   |         |
+-----------+         |
|     x   |-----------+
+-----------+
|    ret    |
+-----------+
|    rbp    |
+-----------+
|           |
+-----------+
|    x      |
+-----------+
|    32     |
+-----------+
|    10     |
+-----------+
|    6754   |
+-----------| <+ rsp
```
综上所述，使用接口来调用方法，会经历两次方法调用步骤，第一次是创建iface后，调用系统自动生成的包装函数，第二次则是在包装函数中调用接口所持动态类型的相应方法。

## 向参数为接口的函数中传递一个satisfied type

```go
package main

import (
	"flag"
	"fmt"
)

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

type Adder1 struct {
	id  int32
	str string
}

//go:noinline
func (adder1 Adder1) Add(a, b int32) int32 { return a + b }

//go:noinline
func (adder1 Adder1) Sub(a, b int64) int64 { return a + b }

func main() {
	a := Adder{id: 6754, str: "test"}
	m := Mather(a)
	//m := Mather(Adder1{id: 6754, str: "test"})
	// This call just makes sure that the interface is actually used.
	// Without this call, the linker would see that the interface defined above
	// is in fact never used, and thus would optimize it out of the final
	// executable.
	ret := m.Add(10, 32)
	fmt.Println(ret) // ignore, just for explain

	a1 := Adder1{id: 6754, str: "test"}

	ok := flag.Bool("ok", false, "are you ok?")

	if *ok {
		receiveInterface(a)
	} else {
		receiveInterface(a1)
	}

}

//go:noinline
func receiveInterface(m Mather) int32 { return m.Add(1, 2) }
```

重点关注`receiveInterface(a)`函数调用。

```go
	0x0131 00305 (main.go:54)	PCDATA	$2, $0
	0x0131 00305 (main.go:54)	CMPB	(AX), $0
	0x0134 00308 (main.go:54)	JEQ	405
	0x0136 00310 (main.go:55)	PCDATA	$0, $1
	0x0136 00310 (main.go:55)	MOVL	$6754, ""..autotmp_13+104(SP)
	0x013e 00318 (main.go:55)	PCDATA	$2, $1
	0x013e 00318 (main.go:55)	LEAQ	go.string."test"(SB), AX
	0x0145 00325 (main.go:55)	PCDATA	$2, $0
	0x0145 00325 (main.go:55)	MOVQ	AX, ""..autotmp_13+112(SP)
	0x014a 00330 (main.go:55)	MOVQ	$4, ""..autotmp_13+120(SP)
	0x0153 00339 (main.go:55)	PCDATA	$2, $1
	0x0153 00339 (main.go:55)	LEAQ	go.itab."".Adder,"".Mather(SB), AX
	0x015a 00346 (main.go:55)	PCDATA	$2, $0
	0x015a 00346 (main.go:55)	MOVQ	AX, (SP)
	0x015e 00350 (main.go:55)	PCDATA	$2, $1
	0x015e 00350 (main.go:55)	PCDATA	$0, $0
	0x015e 00350 (main.go:55)	LEAQ	""..autotmp_13+104(SP), AX
	0x0163 00355 (main.go:55)	PCDATA	$2, $0
	0x0163 00355 (main.go:55)	MOVQ	AX, 8(SP)
	0x0168 00360 (main.go:55)	CALL	runtime.convT2I(SB)
	0x016d 00365 (main.go:55)	PCDATA	$2, $1
	0x016d 00365 (main.go:55)	MOVQ	24(SP), AX
	0x0172 00370 (main.go:55)	MOVQ	16(SP), CX
	0x0177 00375 (main.go:55)	MOVQ	CX, (SP)
	0x017b 00379 (main.go:55)	PCDATA	$2, $0
	0x017b 00379 (main.go:55)	MOVQ	AX, 8(SP)
	0x0180 00384 (main.go:55)	CALL	"".receiveInterface(SB)
	0x0185 00389 (<unknown line number>)	PCDATA	$2, $-2
	0x0185 00389 (<unknown line number>)	PCDATA	$0, $-2
	0x0185 00389 (<unknown line number>)	MOVQ	128(SP), BP
	0x018d 00397 (<unknown line number>)	ADDQ	$136, SP
	0x0194 00404 (<unknown line number>)	RET
	0x0195 00405 (main.go:57)	PCDATA	$2, $0
	0x0195 00405 (main.go:57)	PCDATA	$0, $3
	0x0195 00405 (main.go:57)	MOVL	$6754, ""..autotmp_17+80(SP)
	0x019d 00413 (main.go:57)	PCDATA	$2, $1
	0x019d 00413 (main.go:57)	LEAQ	go.string."test"(SB), AX
	0x01a4 00420 (main.go:57)	PCDATA	$2, $0
	0x01a4 00420 (main.go:57)	MOVQ	AX, ""..autotmp_17+88(SP)
	0x01a9 00425 (main.go:57)	MOVQ	$4, ""..autotmp_17+96(SP)
	0x01b2 00434 (main.go:57)	PCDATA	$2, $1
	0x01b2 00434 (main.go:57)	LEAQ	go.itab."".Adder1,"".Mather(SB), AX
	0x01b9 00441 (main.go:57)	PCDATA	$2, $0
	0x01b9 00441 (main.go:57)	MOVQ	AX, (SP)
	0x01bd 00445 (main.go:57)	PCDATA	$2, $1
	0x01bd 00445 (main.go:57)	PCDATA	$0, $0
	0x01bd 00445 (main.go:57)	LEAQ	""..autotmp_17+80(SP), AX
	0x01c2 00450 (main.go:57)	PCDATA	$2, $0
	0x01c2 00450 (main.go:57)	MOVQ	AX, 8(SP)
	0x01c7 00455 (main.go:57)	CALL	runtime.convT2I(SB)
	0x01cc 00460 (main.go:57)	MOVQ	16(SP), AX
	0x01d1 00465 (main.go:57)	PCDATA	$2, $2
	0x01d1 00465 (main.go:57)	MOVQ	24(SP), CX
	0x01d6 00470 (main.go:57)	MOVQ	AX, (SP)
	0x01da 00474 (main.go:57)	PCDATA	$2, $0
	0x01da 00474 (main.go:57)	MOVQ	CX, 8(SP)
	0x01df 00479 (main.go:57)	CALL	"".receiveInterface(SB)
	0x01e4 00484 (main.go:57)	JMP	389
	0x01e6 00486 (main.go:57)	NOP
	0x01e6 00486 (main.go:39)	PCDATA	$0, $-1
	0x01e6 00486 (main.go:39)	PCDATA	$2, $-1
	0x01e6 00486 (main.go:39)	CALL	runtime.morestack_noctxt(SB)
	0x01eb 00491 (main.go:39)	JMP	0
```

可以看到编译器为两处`receiveInterface()`各自生成了汇编代码。


参考资料：
- [Interfaces and other types](https://golang.org/doc/effective_go.html#interfaces_and_types)
- [Understanding Go Interfaces](https://www.youtube.com/watch?v=F4wUrj6pmSI&feature=youtu.be&t=285)
- [Go Data Structures: Interfaces](https://research.swtch.com/interfaces)
- [Chapter II: Interfaces](https://github.com/teh-cmc/go-internals/blob/master/chapter2_interfaces/README.md)