---
title: "Golang 泛型"
date: 2024-10-05
draft: false
description: "golang 泛型 generic"
tags: ["Go", "后端"]
---
## 传统写法

一个函数兼容多种类型的传统写法一般用`interface`+断言来实现，比如遍历一个数组的函数：

```go
func printArray(arr interface{}) {
	// 类型断言 x.(T) 其实就是判断 T是否实现了 x接口， 如果实现了，就把x的interface类型具体转化为 T类型
	switch t := arr.(type) {
	case []int:
		fmt.Println("int:")
		for _, v := range t {
			fmt.Println(v)
		}
	case []string:
		fmt.Println("string", t)
		for _, v := range t {
			fmt.Println(v)
		}
	}
}
```

或者干脆每种可能的类型都单独写一个方法实现：

```go
func printStringArray(arr interface{}) {
	for _, a := range arr.([]string) {
		fmt.Println(a)
	}
}
func printIntArray(arr interface{}) {
	for _, a := range arr.([]int) {
		fmt.Println(a)
	}
}
```

显然这种每种类型都实现一遍的方式很繁琐。

## 泛型

自从go 1.8 版本之后，go语言就推出泛型，可以以此来简化写法。

泛型中心思想就是 **定义的时候不限定他的类型，让调用者自己去定义类型**

```go
// []T 形式类型
func printArray[T string|int](arr []T) {
	for _, a := range arr {
		fmt.Println(a)
	}
}
```

上面的写法我们在形参里用`T` 来泛指某一个类型(形式类型)，函数名与形参中间有一对中括号`[T string|int]` 来约束 `T` 允许接收的类型，这里`T`允许的类型为 `int` 和`string`

在调用的时候，我们用实际的类型作为参数来调用`printArray`函数：

```go
intArr := []int{1,2,3}
printArray(intArr)
stringArr := []string{"a", "b"}
// stringArr 实际类型
printArray(stringArr)
```

### 内置泛型类型

**any**

表示go里面所有的内置基本类型，等价于`interface{}`

**comparable**

表示go里面所有内置的可比较类型：`int`,`uint`,`float`,`bool`,`struct`,`指针`一切可以比较的类型


这时有的同学会问了，通过go `interface`和`反射` 不也可以达到泛型一样的动态数据处理么？是的，但是反射会有一些副作用：

* 用起来麻烦
* 失去了编译时的类型检查，不仔细写容易出错
* 性能不太理想

> 所以，如果你经常为不同类型写相同逻辑的代码，采用泛型是最佳选择。


### 泛型类型

如果一个变量会有多种类型的情况，平时的写法是声明多个变量，每个变量拥有不同的类型，比如

```go
type s1 int
type s2 float64
var a s1 = []int{1,2,3,4}
var b s2 = []float64{1.0,2.0,3.0}
```

那么有没有方法只变量只声明一次，并且代表多种类型呢？答案是可以的，那就要用上上面的说的泛型：

```go
// 定义变量s[T],不去限定运行时具体使用哪个变量
type s[T int|float64] []T // s[T] T可以是int或float64
```

那么如何使用这个变量呢，上面提到了，泛型核心思想是**定义的时候不限定他的类型，让调用者自己去定义类型**

```go
// 调用者自己去定义类型
var a s[int] = []int{1,2,3} // 调用时设定s[T]中T选取int类型去使用
```

除了slice，其他类型也可以使用泛型来定义：

1. map

    ```go
    type MyMap[Key int|string, VALUE any] map[KEY]VALUE // any代表任何类型

    // 调用
    var m1 MyMap[string,float64] = map[string]float64{
    	"go":9.8
    	"php":9.0
    }
    // 或者这样调用
    m1 := make(MyMap[string]float64)
    m1["go"] = 9.8
    m1["php"] = 9.0
    ```

2. struct

    ```go
    type MyStruct[T int|float32|float64] struct {
    	element []T
    }
    ```

3. 接口

    ```go
    type IPrintData[T int|float32|string] interface{
    	Print(data T)
    }
    ```

4. 泛型通道

    ```go
    type MyChan[T int|string] chan T

    // 调用
    var c  = make(MyChan[int])
    c<-1
    ```

### 泛型函数

#### 成员函数

```go
// 声明自定义类型MySlice, 底层类型是[]T 同时T被约束 int或string
// 注意这不是类型别名，go 1.9开始 类型别名是这样写的 type myint = int
// 泛型不可以起别名
type MySlice[T int|string] []T 

// 定义变量MySlice的成员方法Add
// 这里算是调用了泛型，在使用时需要实例化，所以要用MySlice[T]
func (s MySlice[T]) Add() T{
	var sum T
	for _, v := range s {
		sum += v
	}
	return sum
}

func main() {
	// 泛型调用时需要实例化，指定类型
	var s MySlice[int] = []int{1,2,3,4,5}
	fmt.Println(s.Add())
}
```

#### 普通函数

```go
func Add[T int|string](a, b T) T {
	return a + b
}

func main() {
	Add[int](1,2)
	Add[string]("a", "b")

	// 如果参数是能被推断出类型的基本类型，则可以省略泛型的显示定义类型
	Add(1,2)
	Add("a", "b")
}
```

所以在泛型普通函数，可以自动推倒出泛型类型


### 自定义泛型类型

如果变量要泛型的类型太多怎么办，总不能满屏都写成这样：

```go
func Add[T int|int8|int16|int32|int64](a, b T) T {
	return a + b
}
```

这个时候我们可以自定义泛型类型，把要约束的类型集合归类到一起, 像声明接口一样声明：

```go
type MyInt interface {
	int|int8|int16|int32|int64
}

func Add[T MyInt](a, b T) T {
	return a + b
}
```

#### 衍生类型

那如果后续我有一个新的约束类型，并且这个新类型是某个类型的别名，比如：

```go
type newint int8
```

有两种方法新增，一个是直接约束后面追加：

```go
type MyInt interface {
	int|int8|int16|int32|int64|newint
}
```

另一个方法是在类型前加符号`~`，代表这个类型以及其衍生类型，包括我们在这类型上定义的别名

```go
type MyInt interface {
	int|~int8|int16|int32|int64
} 
```