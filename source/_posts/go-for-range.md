---
title: Go for-range 的奇技淫巧
tags:
  - Backend
  - Golang
categories:
  - Golang
date: 2022-05-20 09:01:26
updated: 2022-05-20 09:01:26
---

# 背景


朋友发了两个代码片段给我看，让我猜输出的内容是啥。具体代码如下：

	// Demo1 
	// 1. 这个循环是否能停下来？
	// 2. 如果能停下来，打印的 arr 内容是什么？
	arr := []int{1, 2, 3}
	for _, v := range arr {
		arr = append(arr, v)
	}
	
	fmt.Println(arr)
	
	
	// Demo2
	// 1. idx 和 value 输出多少？
	// 2. 输出几行？
	str := "你好"
	for idx, v := range str {
		fmt.Printf("idx = %d , value = %c\n", idx, v)
	}
	

不卖关子，先说下第一个`Demo`输出的是：

	[1 2 3 1 2 3]
	
第二个`Demo`输出的是：
	
	idx = 0 , value = 你
	idx = 3 , value = 好

为什么是这样，我们往下看。


# Demo1分析

	arr := []int{1, 2, 3}
	for _, v := range arr {
		arr = append(arr, v)
	}
	

我们先看下[Demo1生成的汇编代码](https://godbolt.org/z/vrzfPz4rz)

	
	main_pc0:
	 ..........................
	main_pc101:
	        MOVQ    CX, "".arr+144(SP)
	        MOVQ    $3, "".arr+152(SP)
	        MOVQ    $3, "".arr+160(SP)
	        MOVQ    CX, ""..autotmp_2+192(SP)
	        MOVQ    $3, ""..autotmp_2+200(SP)
	        MOVQ    $3, ""..autotmp_2+208(SP)
	        MOVQ    $0, ""..autotmp_5+80(SP)  // autotmp_5+80 = 0 , 类似 i:=0
	        MOVQ    ""..autotmp_2+200(SP), DX // 这里设置 DX = 3
	        MOVQ    DX, ""..autotmp_6+72(SP) // autotmp_6+72(SP) = 3
	        JMP     main_pc189
	main_pc189:
	        MOVQ    ""..autotmp_5+80(SP), DX // DX = 0 (DX = i)
	        CMPQ    ""..autotmp_6+72(SP), DX // 比较 3 和 DX (i < 3)
	        JGT     main_pc206  // DX < 3 跳转到 body 模块 , 执行 arr = append(arr, v)
	        JMP     main_pc358  // DX >= 3 循环结束。执行后续打印代码。
	main_pc206:
	 ..........................

	 
从上面汇编代码，我们看出，`for range`的循环次数是固定的`3`次，并不是每次都会去读取`arr`的长度，所以`arr`只会`append`三次，也解释了为什么输出是：

	[1 2 3 1 2 3]

我们再来看下`Go编译器`是怎么对`for range`代码翻译转换的。翻了下[Go编译器源码](https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/range.go#L85)，相关代码如下：


	case types.TARRAY, types.TSLICE:
		if nn := arrayClear(nrange, v1, v2, a); nn != nil {
			base.Pos = lno
			return nn
		}

		// order.stmt arranged for a copy of the array/slice variable if needed.
		ha := a

		hv1 := typecheck.Temp(types.Types[types.TINT])
		hn := typecheck.Temp(types.Types[types.TINT])

		init = append(init, ir.NewAssignStmt(base.Pos, hv1, nil))
		init = append(init, ir.NewAssignStmt(base.Pos, hn, ir.NewUnaryExpr(base.Pos, ir.OLEN, ha)))

		nfor.Cond = ir.NewBinaryExpr(base.Pos, ir.OLT, hv1, hn)
		nfor.Post = ir.NewAssignStmt(base.Pos, hv1, ir.NewBinaryExpr(base.Pos, ir.OADD, hv1, ir.NewInt(1)))

		// for range ha { body }
		if v1 == nil {
			break
		}

		// for v1 := range ha { body }
		if v2 == nil {
			body = []ir.Node{ir.NewAssignStmt(base.Pos, v1, hv1)}
			break
		}

可以看到 `ha := a` 这句代码，`for range`的对象是`Array`或者是`Slice`的时候，会先`Copy`一下这个对象。所以在循环的时候`Append`元素到`Slice`中去，并不会改变循环的次数。

编译器会把`for-range`代码转换成伪代码如下：

	ha := a
	hv1 := 0
	hn := len(ha)
	
	for ; hv1 < hn; hv1++ {
		 // v1, v2 = hv1, ha[hv1]
	    // ...
	}

还有一点要指出的是，`Golang`的`Slice`是`胖指针`，所以值复制的时候不会拷贝所有的数据。只会拷贝`SliceHeader`对应的三个对象

	// SliceHeader is the runtime representation of a slice.
	// It cannot be used safely or portably and its representation may
	// change in a later release.
	// Moreover, the Data field is not sufficient to guarantee the data
	// it references will not be garbage collected, so programs must keep
	// a separate, correctly typed pointer to the underlying data.
	type SliceHeader struct {
		Data uintptr
		Len  int
		Cap  int
	}

# Demo2分析

	str := "你好"
	for idx, v := range str {
		fmt.Printf("idx = %d , value = %c\n", idx, v)
	}

我们也来看下 [Demo2生成的汇编代码](https://godbolt.org/z/saEnf3snd)

我先看下关键的循环相关的代码：

	main_pc50:
	        MOVQ    $6, "".str+104(SP)
	        MOVQ    DX, ""..autotmp_3+112(SP)
	        MOVQ    $6, ""..autotmp_3+120(SP)
	        MOVQ    $0, ""..autotmp_5+64(SP)
	        JMP     main_pc84
	main_pc84:
	        MOVQ    ""..autotmp_5+64(SP), DX  // DX = 0
	        NOP
	        CMPQ    ""..autotmp_3+120(SP), DX // 比较 6 和 DX大小
	        JGT     main_pc108  // 6 > DX 跳转到 108
	        JMP     main_pc440  // 6 <= DX 跳转到 440

这里，我们可以看到，循环停止条件是，DX>=`6`。这个`6`的值是怎么算出来的？

因为`Golang`的源码默认都是用的`UTF-8`编码。[UTF-8（8-bit Unicode Transformation Format）](https://zh.m.wikipedia.org/zh-hans/UTF-8)是一种针对Unicode的可变长度字元编码，也是一种前缀码。它可以用一至四个字节对Unicode字符集中的所有有效编码点进行编码

![origin_img_v2_f862efd1-d827-4812-a3f8-9b3e4cd40chu.jpg](https://upload-images.jianshu.io/upload_images/12321605-9b090629b2e86dfb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

`你好`两个汉字对应的`Unicode`编码如下，一共占用`6`个字节。

	11100100 10111101 10100000 // 你
	11100101 10100101 10111101 // 好

再看下`for`循环的的步长是如何算的，汇编代码如下。

	main_pc172:
	        MOVQ    ""..autotmp_3+120(SP), BX
	        PCDATA  $1, $1
	        CALL    runtime.decoderune(SB)
	        MOVL    AX, ""..autotmp_7+44(SP)
	        MOVQ    BX, ""..autotmp_5+64(SP) 
	        NOP
	        JMP     main_pc194

我们可以看到，`decoderune`函数第二个返回值存到了 `""..autotmp_5+64(SP)` 中，上面会把这个赋值给`DX`，`DX`再去跟`6`比较。

再来看下[decoderune](https://github.com/golang/go/blob/master/src/runtime/utf8.go#L60)这个函数是干什么的，找到`runtime`代码如下：

	// decoderune returns the non-ASCII rune at the start of
	// s[k:] and the index after the rune in s.
	//
	// decoderune assumes that caller has checked that
	// the to be decoded rune is a non-ASCII rune.
	//
	// If the string appears to be incomplete or decoding problems
	// are encountered (runeerror, k + 1) is returned to ensure
	// progress when decoderune is used to iterate over a string.
	func decoderune(s string, k int) (r rune, pos int) {

我们可以知道，这个函数会返回当前字符串`k`之后的`rune`字符和`rune`字符对应的位置。所以`demo`的循环`idx`是`0`和`3`，因为`0`、`3`分别是两个字符的起始位置。

[在编译器源码里面](https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/range.go#L220)也可以看到`for-range`字符串的时候生成的伪代码如下：

	// Transform string range statements like "for v1, v2 = range a" into
	//
	// ha := a
	// for hv1 := 0; hv1 < len(ha); {
	//   hv1t := hv1
	//   hv2 := rune(ha[hv1])
	//   if hv2 < utf8.RuneSelf {
	//      hv1++
	//   } else {
	//      hv2, hv1 = decoderune(ha, hv1)
	//   }
	//   v1, v2 = hv1t, hv2
	//   // original body
	// }


也就解释了为什么`Demo2`输出是

	idx = 0 , value = 你
	idx = 3 , value = 好
