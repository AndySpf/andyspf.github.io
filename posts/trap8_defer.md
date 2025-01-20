---
title: defer引出的go函数调用惯例
author: "spf"
tags: ["踩坑", "golang"]
categories: ["golang"]
date: 2021-01-10
---

## 问题 
defer执行顺序是个老生常谈的问题，其顺序应该如下：    
1. return后如果有可计算表达式则先进行计算
2. 检查是否有defer，如果有则从最后一个defer开始执行
3. 顺序执行完所有defer，执行RET
4. 额外需要注意，如果以传参方式传入defer一个变量，则以复制值的方式传入，值为定义defer时变量的值。    

但是今天又被下面两种情况搞迷糊了，返回值是否匿名导致了结果完全不同：
```golang
func g1() int {
	var b int
	defer func() {
		b++
	}()
	return b
}
// 返回0
func g2() (b int) {
	defer func() {
		b++
	}()
	return b
}
// 返回1
```

## 猜想
在匿名函数返回值中，需要在函数中重新申请地址存放返回值，整个函数里包括defer都是对这个地址的值进行的处理。但是由于go函数的调用惯例是调用者预留被调用函数的返回值空间，且一般（不考虑逃逸分析的情况）是在栈上预定的位子。因此在return时会把这个值放回预留出来的位置上，然后执行defer的逻辑。因此defer虽然把函数内的临时变量改变了，但是已经放到栈上的内容却没有变，调用者最终拿到的也是提前预留在栈上的地址表示的数。   

有名字的返回值情况下：栈中预留位置就是整个被调用者计算并赋值的位置。包括defer函数中的计算。

接下来奉献出plan9的第一次，尝试通过看一下汇编来确定猜想：
+ 匿名返回值情况下：
```golang
//go:noinline
func myFunction() int {
	var myNum int
	defer func() {
		myNum += 7
	}()
	myNum += 3
	return myNum
}

func main() {
	myFunction()
}
```
执行 **go tool compile -N -S -l main.go** 打印出对应汇编语句,选取一些关键的部分如下：
```
//首先是main函数
"".main STEXT size=51 args=0x0 locals=0x10
	......
	0x000f 00015 (test.go:14)	SUBQ	$16, SP      开辟16字节内存
	0x0013 00019 (test.go:14)	MOVQ	BP, 8(SP)    存储基址指针到SP向上数第8-16位，0-8位预留给调用myFunction的返回值用
	0x0018 00024 (test.go:14)	LEAQ	8(SP), BP    
	......
	0x001d 00029 (test.go:15)	CALL	"".myFunction(SB)   调用myFunction
	0x0022 00034 (test.go:16)	MOVQ	8(SP), BP           还原旧的BP
	0x0027 00039 (test.go:16)	ADDQ	$16, SP             SP指针加16字节相当于释放等量内存
	0x002b 00043 (test.go:16)	RET
	......


//接下来看下myFunction函数
"".myFunction STEXT size=162 args=0x8 locals=0x68
	......
	0x0013 00019 (test.go:5)	SUBQ	$104, SP   栈扩张，向下扩张104字节
	0x0017 00023 (test.go:5)	MOVQ	BP, 96(SP)   改变函数基址指针位置
	0x001c 00028 (test.go:5)	LEAQ	96(SP), BP
	......
	0x0021 00033 (test.go:5)	MOVQ	$0, "".~r0+112(SP)  先初始化返回值(104+8)
	0x002a 00042 (test.go:6)	MOVQ	$0, "".myNum+8(SP)  初始化声明的局部变量myNum
	......     这一段是defer相关的汇编逻辑，比如将_defer加入链表内，初始化等工作，这里的defer对应的是deferprocStack会在栈上创建defer     
	0x006a 00106 (test.go:10)	ADDQ	$3, AX           AX寄存器加3
	0x006e 00110 (test.go:10)	MOVQ	AX, "".myNum+8(SP)   将AX寄存器的值给myNum
	0x0073 00115 (test.go:11)	MOVQ	AX, "".~r0+112(SP)   将AX寄存器的值给返回值
	......
	0x0067 00103 (test.go:10)   CALL    runtime.deferreturn(SB)   // 开始执行defer的逻辑,可以证明的确是return赋值后，ret前进行的defer逻辑。defer的执行其实就类似于函数内执行一个内部定义的函数，只是执行的时间和编译方式不同。
	......
```
可以看到对应的汇编实现里其实额外创建了一个不可见的变量，return时仅仅是把结果赋值给了这个临时变量，之后defer的操作就和返回值那个临时变量无关了。

+ 命名返回值情况下：
```golang
//go:noinline
func myFunction() (myNum int) {
	defer func() {
		myNum += 7
	}()
	myNum += 3
	return myNum
}

func main() {
	myFunction()
}
```
其对应的myFunction段汇编为:
```
"".myFunction STEXT size=144 args=0x8 locals=0x60
	......
	0x000f 00015 (test.go:4)	SUBQ	$96, SP
	......
	0x001d 00029 (test.go:4)	MOVQ	$0, "".myNum+104(SP)  返回值被命名为myNum，也的确是对(96+8)的栈地址，即main函数预留出的返回值地址进行的初始化
	......
	0x0058 00088 (test.go:9)	MOVQ	"".myNum+104(SP), AX
	0x005d 00093 (test.go:9)	ADDQ	$3, AX
	0x0061 00097 (test.go:9)	MOVQ	AX, "".myNum+104(SP)
	......
```
这里可以看到，如果命名返回值，则省掉了额外声明返回值变量的栈分配。defer函数内访问到的就是返回值的地址。

基本证明猜想是正确的


