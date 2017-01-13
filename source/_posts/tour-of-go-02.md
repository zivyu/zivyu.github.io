---
title: A Tour of Go（二）：方法和接口
date: 2017-1-3 19:27:26
description: 本节我们主要学习怎么定义方法、怎么声明接口，等。Go语言没有类，但是我们可以在类型上定义方法。方法有一个特殊的接受者参数的函数。...
categories:
- Go语言
tags:
- Go 
- 编程语言
---

本节我们主要学习怎么定义方法、怎么声明接口，等。

### 方法

Go语言没有类，但是我们可以在类型上定义方法。

方法有一个特殊的*接受者*参数的函数。

接受者出现在`func`关键字和方法名之间的参数列表中。

比如说下面的例子，方法`Abs`有一个类型为`Vertex`的接受者。

``` go
package main

import (
	"fmt"
	"math"
)

type Vertex struct {
	X, Y float64
}

func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := Vertex{3, 4}
	fmt.Println(v.Abs())
}
```

### 方法及函数

方法仅仅是一个有接收者参数的函数。

可以把方法重写为一个函数，如下乳，`Abs`可以被写为一个常规的方法

``` go
package main

import (
	"fmt"
	"math"
)

type Vertex struct {
	X, Y float64
}

func Abs(v Vertex) float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := Vertex{3, 4}
	fmt.Println(Abs(v))
}
```

### 方法（续）

也可以为non-struct类型声明方法，

看下面的例子，数值类型`MyFloat`有一个`Abs`方法。

``` go
package main

import (
	"fmt"
	"math"
)

type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

func main() {
	f := MyFloat(-math.Sqrt2)
	fmt.Println(f.Abs())
}
```

### 指针接收者

可以声明方法的接收者是指针。

看下面的例子，`Scale`方法定义在`*Vertex`之上。

这样可以修改接收者的值，如果不传指针，则是原始对象的一个拷贝。

``` go 
package main

import (
	"fmt"
	"math"
)

type Vertex struct {
	X, Y float64
}

func (v Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func main() {
	v := Vertex{3, 4}
	v.Scale(10)
	fmt.Println(v.Abs())
}
```

### 指针和函数

也可以把方法重写成一个函数：

``` go
package main

import (
	"fmt"
	"math"
)

type Vertex struct {
	X, Y float64
}

func Abs(v Vertex) float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func Scale(v Vertex, f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func main() {
	v := Vertex{3, 4}
	Scale(v, 10)
	fmt.Println(Abs(v))
}
```

### 方法和间接指针

在之前的例子中，有一个指针参数的方法比如传入一个指针：

	var v Vertex
	ScaleFunc(v)  // Compile error!
	ScaleFunc(&v) // OK

而调用指针接收者的方法的时候，既不需要传入值也不需要传入指针：

	var v Vertex
	v.Scale(5)  // OK
	p := &v
	p.Scale(10) // OK

对于语句`v.Scale(5)`，会自动调用有指针接收者的方法。比如说，`v.Scale(5)`会变为`(&v).Scale(5)`，因为`Scale`方法有一个指针接收者：

``` go
package main

import "fmt"

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func ScaleFunc(v *Vertex, f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func main() {
	v := Vertex{3, 4}
	v.Scale(2)
	ScaleFunc(&v, 10)

	p := &Vertex{4, 3}
	p.Scale(3)
	ScaleFunc(p, 8)

	fmt.Println(v, p)
}
```

### 选择值接收者还是指针接收者

选择指针接收者有两个原因：

第一个原因是可以修改接收者指针的值。

第二个原因避免每次调用的时候复制值。

``` go
package main

import (
	"fmt"
	"math"
)

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Scale(f float64) {
	v.X = v.X * f
	v.Y = v.Y * f
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}

func main() {
	v := &Vertex{3, 4}
	fmt.Printf("Before scaling: %+v, Abs: %v\n", v, v.Abs())
	v.Scale(5)
	fmt.Printf("After scaling: %+v, Abs: %v\n", v, v.Abs())
}
```

### 接口

*接口类型*是一组方法签名的集合。

接口类型的值可以存放实现这些方法的任何值。

``` go
package main

import (
	"fmt"
	"math"
)

type Abser interface {
	Abs() float64
}

func main() {
	var a Abser
	f := MyFloat(-math.Sqrt2)
	v := Vertex{3, 4}

	a = f  // a MyFloat implements Abser
	a = &v // a *Vertex implements Abser

	// In the following line, v is a Vertex (not *Vertex)
	// and does NOT implement Abser.
	a = v

	fmt.Println(a.Abs())
}

type MyFloat float64

func (f MyFloat) Abs() float64 {
	if f < 0 {
		return float64(-f)
	}
	return float64(f)
}

type Vertex struct {
	X, Y float64
}

func (v *Vertex) Abs() float64 {
	return math.Sqrt(v.X*v.X + v.Y*v.Y)
}
```

### 接口的隐式实现

一个类型通过实现接口的方法来实现一个接口。不需要显示的声明，也没有"implements"关键字。

``` go
package main

import "fmt"

type I interface {
	M()
}

type T struct {
	S string
}

// This method means type T implements the interface I,
// but we don't need to explicitly declare that it does so.
func (t T) M() {
	fmt.Println(t.S)
}

func main() {
	var i I = T{"hello"}
	i.M()
}
```

### 空接口

空接口没有指定任何方法：

	interface{}

一个空接口可以存储任何类型的值。

空接口用来编写处理未知类型的值。比如说`fmt.Print` 处理许多`interface{}`的参数：

``` go 
package main

import "fmt"

func main() {
	var i interface{}
	describe(i)

	i = 42
	describe(i)

	i = "hello"
	describe(i)
}

func describe(i interface{}) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

### 类型断言

*类型断言*

	t := i.(T)

上述语言断言接口值`i`保存了具体的类型`T`，并把具体`T`的值分配给变量`t`。

但是如果`i`没有保存一个`T`，上述的语句会出现错误。

为了`测试`一个接口值是否保存了一个具体的类型，类型断言可以返回两个值：具体的类型值和断言是否成功。

	t, ok := i.(T)

如果`i`保存着`T`，然后`t`将会是具体的值，`ok`将会是true。

反之，`ok`将是false，`t`将会是类型`T`的零值。

``` go
package main

import "fmt"

func main() {
	var i interface{} = "hello"

	s := i.(string)
	fmt.Println(s)

	s, ok := i.(string)
	fmt.Println(s, ok)

	f, ok := i.(float64)
	fmt.Println(f, ok)

	f = i.(float64) // panic
	fmt.Println(f)
}
```

### 类型switches

类型转换就像常规的转换语句一样，但是case是具体的类型而不是值。

	switch v := i.(type) {
	case T:
	    // here v has type T
	case S:
	    // here v has type S
	default:
	    // no match; here v has the same type as i
	}

### Stringers

最普通的接口之一是`Stringer`

	type Stringer interface {
	    String() string
	}

一个`Stringer`是一个可以把它自己描述为一个字符串的类型。`fmt`包寻找这个接口来打印值。

``` go
package main

import "fmt"

type Person struct {
	Name string
	Age  int
}

func (p Person) String() string {
	return fmt.Sprintf("%v (%v years)", p.Name, p.Age)
}

func main() {
	a := Person{"Arthur Dent", 42}
	z := Person{"Zaphod Beeblebrox", 9001}
	fmt.Println(a, z)
}
```

### 错误

Go程序使用`error`值来表达错误状态。

`error`类型是内建接口。

	type error interface {
	    Error() string
	}

函数经常返回一个`error`值，调用程序通过调用error是否等于`nil`来处理错误。

	i, err := strconv.Atoi("42")
	if err != nil {
	    fmt.Printf("couldn't convert number: %v\n", err)
	    return
	}
	fmt.Println("Converted integer:", i)

一个nil的`error`表示成功，一个non-nil的`error`表示失败。

``` go
package main

import (
	"fmt"
	"time"
)

type MyError struct {
	When time.Time
	What string
}

func (e *MyError) Error() string {
	return fmt.Sprintf("at %v, %s",
		e.When, e.What)
}

func run() error {
	return &MyError{
		time.Now(),
		"it didn't work",
	}
}

func main() {
	if err := run(); err != nil {
		fmt.Println(err)
	}
}
```