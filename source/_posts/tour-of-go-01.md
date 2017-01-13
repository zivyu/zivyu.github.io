---
title: A Tour of Go（一）：基本知识
date: 2016-12-30 10:29:39
categories:
- Go语言
tags:
- Go 
- 编程语言
---

在正式开始Mit 6.824之前，我先按照[A Tour of Go](https://golang.org/doc/#go_tour) ，学习Go语言的语法和相关知识。

A Tour of Go分为三个章节。第一个章节描述了基本的语法和数据结构，第二个章节讨论了方法和接口，第三个章节介绍了Go语言的并发原语。

本节描述了Go语言的一些基本知识。包括变量声明，方法调用，以及一些其他的知识。

<!-- more -->

## Hello,world!

``` go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```

## 包，变量和方法

### 包

每个Go程序都由包组成。程序由`main`包处开始运行。比如说下面的程序使用导入的包"fmt"和"math/rand"。

``` go
package main

import (
	"fmt"
	"math/rand"
)

func main() {
	fmt.Println("My favorite number is", rand.Intn(10))
}
```

### 导入

略

### Exported names

在Go中，如果一个单词是以大写字母开头，它就是exported name。比如说Pizza是一个exported name，Pi是一个来自`math`包的exported name。

如下图，直接使用`math`包中的Pi

``` go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println(math.Pi)
}
```

### 方法

一个方法可以有一个或多个参数。

注意类型在变量名之后。

``` go
package main

import "fmt"

func add(x int, y int) int {
	return x + y
}

func main() {
	fmt.Println(add(42, 13))
}
```
当两个或多个参数都是同一个类型的时候，可以只保留最后一个类型。

比如说把
``` go
x int, y int
```
写成
``` go
x, y int
```

### 多个返回结果

一个方法可以返回多个结果。如下，swap返回两个字符串。
``` go
package main

import "fmt"

func swap(x, y string) (string, string) {
	return y, x
}

func main() {
	a, b := swap("hello", "world")
	fmt.Println(a, b)
}
```
### 命令返回值

Go的返回值可以被命令，就像在方法中命令变量一样。如下图，命令了返回值为x和y.

``` go
package main

import "fmt"

func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}

func main() {
	fmt.Println(split(17))
}
```

### 变量

可以用`var`来声明一个变量列表。类型在后面。

``` go
package main

import "fmt"

var c, python, java bool

func main() {
	var i int
	fmt.Println(i, c, python, java)
}
```

### 变量初始化

一个`var`声明可以包含初始化，每个变量对应一个。

如果变量被初始化了，类型可以省略，

``` go
package main

import "fmt"

var i, j int = 1, 2

func main() {
	var c, python, java = true, false, "no!"
	fmt.Println(i, j, c, python, java)
}
```

### 声明短变量

在一个方法中，`:=`语句可以代替一个`var`声明。而在方法之外，`:=`是不可用的。

``` go
package main

import "fmt"

func main() {
	var i, j int = 1, 2
	k := 3
	c, python, java := true, false, "no!"

	fmt.Println(i, j, k, c, python, java)
}
```

### 基本类型

Go的基本类型有：

	bool

	string

	int  int8  int16  int32  int64
	uint uint8 uint16 uint32 uint64 uintptr

	byte 

	rune 

	float32 float64

	complex64 complex128

`int`,`uint`,`uintptr`的宽度和系统是一样的。当你需要一个整型值，应该使用`int`。除非有特别的理由来使用一个定长的或无符号的整数类型。


### 零值

当变量被声明而没被显示的初始化的时候，这些变量会被初始化为*零值*。

数值类型为`0`

布尔类型为`false`

字符串为`""`

### 类型转换

表达式`T(v)`将`v`转换为类型`T`。

一些数值类型的转换：

	var i int = 42
	var f float64 = float64(i)
	var u uint = uint(f)

也可以有更简单的写法：

	i := 42
	f := float64(i)
	u := uint(f)


### 类型推测

当声明一个变量而没有显示的指定类型的时候，会从右侧的值推测出变量的类型。

当右边的值已经被指定类型，这个新的变量就是同样的类型。

	var i int
	j := i // j is an int

但是右边是没有类型的数值常量，新的变量可能是`int`,`float64`,或`complex128`，这取决于常量的精度：

	i := 42           // int
	f := 3.142        // float64
	g := 0.867 + 0.5i // complex128

### 常量

常量声明和普通变量一样，但是要带上`const`关键字。

常量可以是字符、字符串、布尔类型、或数值类型的值。

常量声明不能使用`:=`语法。

### 数值常量

数值常量是高精度的值。

一个没有类型的常量会根据上下文来决定类型。

## 流程控制语句

### For

Go语言只有一个循环结构，就是`for`循环。

for循环有三个用分号隔开的成分：

	初始化语句：在第一次开始循环之前被执行。

	条件表达式：每次开始循环的时候被求值。

	post语句：每次循环结束的时候执行。

一旦条件表达式是`false`，循环将会停止。

``` go
package main

import "fmt"

func main() {
	sum := 0
	for i := 0; i < 10; i++ {
		sum += i
	}
	fmt.Println(sum)
}
```

初始化和post语句都是可以被省略的：

``` go
package main

import "fmt"

func main() {
	sum := 1
	for ; sum < 1000; {
		sum += sum
	}
	fmt.Println(sum)
}
```

### Go中的"while"

当去掉分号的时候，`for`就是C语言中的`while`。

``` go
package main

import "fmt"

func main() {
	sum := 1
	for sum < 1000 {
		sum += sum
	}
	fmt.Println(sum)
}
```

### 无限循环

当省略掉条件表达式的时候，for循环就变成了无限循环。

``` go
package main

func main() {
	for {
	}
}
```

### If语句

Go中if和for循环差不多，表达式都不需要被"()"括起来。

``` go
package main

import (
	"fmt"
	"math"
)

func sqrt(x float64) string {
	if x < 0 {
		return sqrt(-x) + "i"
	}
	return fmt.Sprint(math.Sqrt(x))
}

func main() {
	fmt.Println(sqrt(2), sqrt(-4))
}
```

### If中的表达式

像`For`一样，`if`可以在执行判断条件之前执行一个简单的语句。

``` go
package main

import (
	"fmt"
	"math"
)

func pow(x, n, lim float64) float64 {
	if v := math.Pow(x, n); v < lim {
		return v
	}
	return lim
}

func main() {
	fmt.Println(
		pow(3, 2, 10),
		pow(3, 3, 20),
	)
}
```

### if and else

在`if`中声明的变量，在`else`中也有用。

``` go
package main

import (
	"fmt"
	"math"
)

func pow(x, n, lim float64) float64 {
	if v := math.Pow(x, n); v < lim {
		return v
	} else {
		fmt.Printf("%g >= %g\n", v, lim)
	}
	// can't use v here, though
	return lim
}

func main() {
	fmt.Println(
		pow(3, 2, 10),
		pow(3, 3, 20),
	)
}
```

### Switch 以及 Switch的执行顺序

Switch 从上到下依次执行，当条件满足的时候停止。

比如说：

	switch i {
	case 0:
	case f():
	}

如果i==0，不会调用f。

### 没有条件的Switch

没有条件的Switch就和`Switch true`是一样的。

### Defer

一个defer语句推迟方法的执行直到方法返回。

``` go
package main

import "fmt"

func main() {
	defer fmt.Println("world")

	fmt.Println("hello")
}
```

### defer栈

defer方法调用被压入到栈中，当方法返回的时候，defer调用按照后进先出的顺序被执行。

``` go
package main

import "fmt"

func main() {
	fmt.Println("counting")

	for i := 0; i < 10; i++ {
		defer fmt.Println(i)
	}

	fmt.Println("done")
}
```

## 更多的类型：structs，slices，和maps

### 指针

Go拥有指针。一个指针是一个变量的内存地址。

类型`*T`是指向值`T`的一个指针。它的零值是`nil`。

	var p *int

`&`操作符会生成一个指向它的操作数的指针。

	i := 42
	p = &i

`*`操作符表示指针的底层的值。

	fmt.Println(*p) // 通过指针p读取i的值
	*p = 21         // 通过指针p给i设置值

但是Go没有指针运算。

### 结构体

一个`struct`是字段的一个集合。

``` go
package main

import "fmt"

type Vertex struct {
	X int
	Y int
}

func main() {
	fmt.Println(Vertex{1, 2})
}
```

### 结构体字段

通过点号来访问结构体字段。

``` go
package main

import "fmt"

type Vertex struct {
	X int
	Y int
}

func main() {
	v := Vertex{1, 2}
	v.X = 4
	fmt.Println(v.X)
}
```

### 指向结构体的指针

可以通过结构体指针来访问结构体字段。

可以通过`(*p).X`来访问一个结构体的字段`X`。但是这样写很笨重，所以可以直接写`p.X`。

``` go
package main

import "fmt"

type Vertex struct {
	X int
	Y int
}

func main() {
	v := Vertex{1, 2}
	p := &v
	p.X = 1e9
	fmt.Println(v)
}
```

### 数组

类型`[n]T`是一个有n个类型为T的值的数组。

表达式像下面这样：

	var a [10]int

上面的语法声明有10个整型的数组。

数组不能改变大小。

``` go
package main

import "fmt"

func main() {
	var a [2]string
	a[0] = "Hello"
	a[1] = "World"
	fmt.Println(a[0], a[1])
	fmt.Println(a)

	primes := [6]int{2, 3, 5, 7, 11, 13}
	fmt.Println(primes)
}
```

### Slices

Slices是一个固定大小的数组。另一方面，slice是一个数组的动态大小的、灵活的视图。实际上，Slices比数组更通用。

类型`[]T`是类型T的一个元素的一个slice。

下面的表达式创建了数组a的前五个元素的一个slice：

	a[0:5]

``` go
package main

import "fmt"

func main() {
	primes := [6]int{2, 3, 5, 7, 11, 13}

	var s []int = primes[1:4]
	fmt.Println(s)
}
```

### Slices 就像是数组的引用

一个slice本身不存储任何数据，仅仅描述了一个数组的一部分。

更新一个slice的元素相当于更新数组。

``` go
package main

import "fmt"

func main() {
	names := [4]string{
		"John",
		"Paul",
		"George",
		"Ringo",
	}
	fmt.Println(names)

	a := names[0:2]
	b := names[1:3]
	fmt.Println(a, b)

	b[0] = "XXX"
	fmt.Println(a, b)
	fmt.Println(names)
}
```

### 默认Slice 

当进行切片的时候，可以省略最高和最低的界限值。

对于数组

	var a [10]int

下面的表达式都是相等的：

	a[0:10]
	a[:10]
	a[0:]
	a[:]

### Slice的长度和容量

一个slice既有*长度*和*容量*。

长度就是slice包含的元素个数。

容量是数组元素的个数，从slice中的第一个元素开始计数。

一个slice`S`的长度和容量可以使用表达式`len(s)`和`cap(s)`来获取。

``` go
package main

import "fmt"

func main() {
	s := []int{2, 3, 5, 7, 11, 13}
	printSlice(s)

	// Slice the slice to give it zero length.
	s = s[:0]
	printSlice(s)

	// Extend its length.
	s = s[:4]
	printSlice(s)

	// Drop its first two values.
	s = s[2:]
	printSlice(s)
}

func printSlice(s []int) {
	fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
}
```

### Nil slices

一个slice的零值是`nil`。

一个nil slice的长度和容量为0，没有对应的数组。

``` go
package main

import "fmt"

func main() {
	var s []int
	fmt.Println(s, len(s), cap(s))
	if s == nil {
		fmt.Println("nil!")
	}
}
```

### 使用make方法创建一个slice

可以使用`make`方法创建Slices，这就是创建一个动态大小数组的方式。

`make`方法分配一个零值的数组，然后返回一个slice指向这个数组。

	a := make([]int, 5)  // len(a)=5

传入第三个参数来指定容量：

	b := make([]int, 0, 5) // len(b)=0, cap(b)=5

	b = b[:cap(b)] // len(b)=5, cap(b)=5
	b = b[1:]      // len(b)=4, cap(b)=4

### Slices of slices

Slices可以包含任何类型，也可以包含其他的slices。

``` go
package main

import (
	"fmt"
	"strings"
)

func main() {
	// Create a tic-tac-toe board.
	board := [][]string{
		[]string{"_", "_", "_"},
		[]string{"_", "_", "_"},
		[]string{"_", "_", "_"},
	}

	// The players take turns.
	board[0][0] = "X"
	board[2][2] = "O"
	board[1][2] = "X"
	board[1][0] = "O"
	board[0][2] = "X"

	for i := 0; i < len(board); i++ {
		fmt.Printf("%s\n", strings.Join(board[i], " "))
	}
}
```

### 向一个slice添加元素

追加一个新元素到slice是很普遍的，所以Go提供了一个`apend`方法：

	func append(s []T, vs ...T) []T

第一个参数`s`是一个slice，剩下的参数是追加到slice中的值。

``` go
package main

import "fmt"

func main() {
	var s []int
	printSlice(s)

	// append works on nil slices.
	s = append(s, 0)
	printSlice(s)

	// The slice grows as needed.
	s = append(s, 1)
	printSlice(s)

	// We can add more than one element at a time.
	s = append(s, 2, 3, 4)
	printSlice(s)
}

func printSlice(s []int) {
	fmt.Printf("len=%d cap=%d %v\n", len(s), cap(s), s)
}
```

### Range

`for`循环的`range`格式可以迭代一个slice或map。

当rangey一个slice，每次迭代都会返回两个值，第一个是index，第二个是该index位置元素的副本。

``` go
package main

import "fmt"

var pow = []int{1, 2, 4, 8, 16, 32, 64, 128}

func main() {
	for i, v := range pow {
		fmt.Printf("2**%d = %d\n", i, v)
	}
}
```

### Range continued

通过分配`_`来跳过index或value。

如果你仅仅只是需要index，去掉", value"。

``` go
package main

import "fmt"

func main() {
	pow := make([]int, 10)
	for i := range pow {
		pow[i] = 1 << uint(i) // == 2**i
	}
	for _, value := range pow {
		fmt.Printf("%d\n", value)
	}
}
```

### Maps

一个map是keys到value的映射。

map的零值是nil。nil的map没有keys，也不能添加key。

`make`方法返回一个map。

``` go
package main

import "fmt"

type Vertex struct {
	Lat, Long float64
}

var m map[string]Vertex

func main() {
	m = make(map[string]Vertex)
	m["Bell Labs"] = Vertex{
		40.68433, -74.39967,
	}
	fmt.Println(m["Bell Labs"])
}
```

### Map操作

插入或更新map中的一个元素：

	m[key] = elem

获取一个元素：

	elem = m[key]

删除一个元素：

	delete(m, key)

测试一个key是否存在：

	elem, ok = m[key]

如果key存在，ok为true，如果不存在，ok为false。

如果key不存在，elem为零值。

``` go
package main

import "fmt"

func main() {
	m := make(map[string]int)

	m["Answer"] = 42
	fmt.Println("The value:", m["Answer"])

	m["Answer"] = 48
	fmt.Println("The value:", m["Answer"])

	delete(m, "Answer")
	fmt.Println("The value:", m["Answer"])

	v, ok := m["Answer"]
	fmt.Println("The value:", v, "Present?", ok)
}
```

### 方法值

方法也是一个值。可以像其他值一样被传递。

``` go
package main

import (
	"fmt"
	"math"
)

func compute(fn func(float64, float64) float64) float64 {
	return fn(3, 4)
}

func main() {
	hypot := func(x, y float64) float64 {
		return math.Sqrt(x*x + y*y)
	}
	fmt.Println(hypot(5, 12))

	fmt.Println(compute(hypot))
	fmt.Println(compute(math.Pow))
}
```

### 闭包

Go的方法可以是一个闭包，闭包是一个方法值，引用了方法体之外的值。这个方法可以读取和分配值给这个引用的变量。换句话说，方法被绑定在变量上。

如下的例子，`adder`方法返回一个闭包。每个闭包绑定在它自己的变量`sum`上。

``` go
package main

import "fmt"

func adder() func(int) int {
	sum := 0
	return func(x int) int {
		sum += x
		return sum
	}
}

func main() {
	pos, neg := adder(), adder()
	for i := 0; i < 10; i++ {
		fmt.Println(
			pos(i),
			neg(-2*i),
		)
	}
}
```

