---
title: 'Golang String'
tags:
    - Golang
date: '2022-11-20'
categories:
draft: false
---

## 结构

```go
type _string struct {
	elements *byte // 引用着底层的字节
	len      int   // 字符串中的字节数
}
```

从这个声明来看，我们可以将一个字符串的内部定义看作为一个字节序列。 事实上，我们确实可以把一个字符串看作是一个元素类型为 `byte` 的（且元素不可修改的）切片。

注意，前面的文章已经提到过多次，`byte` 是内置类型 `uint8` 的一个别名。

## 字符串编码和Unicode码点

Unicode标准为全球各种人类语言中的每个字符制定了一个独一无二的值。 但Unicode标准中的基本单位不是字符，而是码点（code point）。大多数的码点实际上就对应着一个字符。 但也有少数一些字符是由多个码点组成的。

码点值在Go中用rune值来表示。 内置rune类型为内置int32类型的一个别名。

字符串和字节切片之间的转换的编译器优化

## 使用`for-range`循环遍历字符串中的码点

`for-range`循环控制中的`range`关键字后可以跟随一个字符串，用来遍历此字符串中的码点（而非字节元素）。 字符串中非法的UTF-8编码字节序列将被解读为Unicode替换码点值`0xFFFD`。

那么如何遍历一个字符串中的字节呢？使用传统`for i := 0; i < len(s); i++`循环：

## 字符串衔接方法

除了使用`+`运算符来衔接字符串，我们也可以用下面的方法来衔接字符串：

- fmt标准库包中的`Sprintf`/`Sprint`/`Sprintln`函数可以用来衔接各种类型的值的字符串表示，当然也包括字符串类型的值。
- 使用`strings.Join`。
- `bytes.Buffer`类型可以用来构建一个字节切片，然后我们可以将此字节切片转换为一个字符串。
- 从Go 1.10开始，`strings.Builder`类型可以用来拼接字符串。 和`bytes.Buffer`类型类似，此类型内部也维护着一个字节切片，但是它在将此字节切片转换为字符串时避免了底层字节的深复制。

标准编译器对使用`+`运算符的字符串衔接做了特别的优化。 所以，一般说来，**在被衔接的字符串的数量是已知的情况下**，使用`+`运算符进行字符串衔接是比较高效的。


### 字符串的比较

上面已经提到了比较两个字符串事实上逐个比较这两个字符串中的字节。 Go 编译器一般会做出如下的优化：
- 对于`==`和`!=`比较，如果这两个字符串的长度不相等，则这两个字符串肯定不相等（无需进行字节比较）。
- 如果这两个字符串底层引用着字符串切片的指针相等，则比较结果等同于比较这两个字符串的长度。

所以两个相等的字符串的比较的时间复杂度取决于它们底层引用着字符串切片的指针是否相等。 如果相等，则对它们的比较的时间复杂度为`O(1)`，否则时间复杂度为`O(n)`。

上面已经提到了，对于标准编译器，一个字符串赋值完成之后，目标字符串和源字符串将共享同一个底层字节序列。 所以比较这两个字符串的代价很小。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	bs := make([]byte, 1<<26)
	s0 := string(bs)
	s1 := string(bs)
	s2 := s1

	// s0、s1和s2是三个相等的字符串。
	// s0的底层字节序列是bs的一个深复制。
	// s1的底层字节序列也是bs的一个深复制。
	// s0和s1底层字节序列为两个不同的字节序列。
	// s2和s1共享同一个底层字节序列。

	startTime := time.Now()
	_ = s0 == s1
	duration := time.Now().Sub(startTime)
	fmt.Println("duration for (s0 == s1):", duration)

	startTime = time.Now()
	_ = s1 == s2
	duration = time.Now().Sub(startTime)
	fmt.Println("duration for (s1 == s2):", duration)
}
```