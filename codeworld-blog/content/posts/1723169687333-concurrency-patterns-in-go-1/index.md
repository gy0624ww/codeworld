---
title: "Go并发模式（一）"
date: 2019-11-18
draft: false
description: "a description"
disqusIdentifier: "Go并发模式（一）"
tags: ["Go", "后端"]
showComments: true
---

## 前言  
  不知道大家有没有过，基础语法明明已经掌握了，但是一看别人的源码还是一头雾水，总有一种我和他学的不是同一门语言的赶脚，有没有看到别人的思路总是感觉那么新奇，写法那么的独特，困惑的同时又羡慕呢？

  其实这个和其他语言一样是存在设计模式的

{{< alert icon="circle-info">}}
所谓设计模式, 就是大家经过一段时间实践并总结出来的一种解决问题的方法模式
{{< /alert >}}

  然而这个设计模式，就像武功秘籍一样，分不同的招式，来应对不同场景的问题，大家只要领会了招式的内涵，加上勤加苦练，相信大家也会写出一手优美的代码来的

下面我们来进入正题啦, 下面我会把一些常见的模式归纳出来，有的会比较基础且常见，有的会彼此组合来使得扩展代码更加强壮。  


## Close-Channel

我们知道当channel被关闭的时候，我们可以从中读取到:

```go
intStream := make(chan int)
close(intStream)
integer, ok := <- intStream // 我们能从close channel 读取到值,ok=false,并且channel里类型的默认值
fmt.Printf("(%v):%v", ok, integer)
// output: (false):0
```

我们并没有往channel里写入任何东西，当我们关闭channel之后我们依旧可以不断的从channel中读取，下面的一些模式里我们会经常用到这一特性

比如这样：

```go
intStream := make(chan int)
go func() {
  defer close(intStream)
  for i:=1;i<=5;i++ {
    intStream <-i
  }
}

for integer := range intSteam {
  fmt.Printf("%v ", integer)
}
```

结果输出并程序退出：

```
1 2 3 4 5
```

上面的例子里我们确保我我们执行完goroutine之后关闭了`intStream`, 并且提供了循环的终止条件，这是非常常见的模式
  

{{< alert icon="circle-info">}}
关闭channel也是同时通知多个`goroutine`方法之一。
{{< /alert >}}

当n个`goroutine`等待同一个channel,block的时候，与其向channel写入n次数据来unblock`goroutine`，不如简单的去关闭这个channel,因为我们可以从已经关闭的channel里无限次的读取数据，不管有多少个`goroutine`在等待。  
并且啊，关闭channel这个方法比写入n次数据的做法成本低又快速。 我们来看下面的例子：

```go
begin := make(chan interface{})
var wg sync.WaitGroup
for i:=0;i<5;i++ {
  wg.Add(1)
  go func(i int) {
    defer wg.Done()
    <-begin // goroutine blocked，直到被告知可以继续
    fmt.Printf("%v has begun\n", i)
  }(i)
}
fmt.Println("Unblocking goroutines...")
close(begin)  // 同时unblock所有goroutine
wg.Wait()
```

输出：
```
Unblocking goroutines...
4 has begun
2 has begun
3 has begun
0 has begun
1 has begun
```
我们可以看到所有的`goroutine`都没有开始执行，直到先打印输出之后，我们关闭了channel。

## channel约束 

{{< figure
    src="https://s2.ax1x.com/2020/01/03/laKv6I.png"
    alt="Result of channel operations given a channel's state"
    caption="Result of channel operations given a channel's state"
    >}}

从这个表格可以看出来和`channel`交互的情形中有3种操作会导致一个`goroutine` block, 有三种操作会到只程序`panic`, 乍一看会感觉操作`channel` 有点危险，但是正因为我们把所有会遇到的情形归纳出来了，牢牢记住，反而不会再担心会踩坑了。
下面我们就来看看如何组织`channel`来实现一些健壮稳定的代码  

首先我们要先分配`channel`的`物主身份`，这个`物主身份`是指某个实例化，写入并且关闭一个`channel`的`goroutine`,为了使得代码逻辑明确，很有必要明确哪个`goroutine`拥有的`channel`.  
单向`channel`是一个很好的工具来区分`channel`是自己拥有的还是外面传进来的：    
因为`channel`的拥有者有对`channel`的写权限（`chan` or `chan<-`）  
而`channel`的使用者仅仅对`channel`拥有读权限（`<-chan`）  
一旦我们对`channel`的所有者和非`channel`的所有者之间进行了区分，上面表格的结果自然而然就会得到，我们就可以依此来分别对`channel`的所有者和非所有者进行责任划分。  

首先我们先来确定`channel`所有者的责任划分，所拥有`channel`的`goroutine`应该满足下面4点：

1. 实例化一个`channel`
2. 执行写操作或者将`物主身份`传递给另一个`goroutine`  
3. 负责关闭该`channel`  
4. 将上面3条打包并通过一个只读`channel`暴露给外界  

通过上面的对`channel`进行责任划分，将会达到下面的效果： 

1. 正因为是我们初始化的`channel`, 所以我们避免了往`nil channel`写入数据导致`dead lock`的风险
2. 正因为是我们初始化的`channel`, 所以我们避免了由于关闭`nil channel`产生的`panic`的风险
3. 正因为是我们来决定`channel`何时关闭， 所以我们避免了由于往`closed channel`继续写入数据产生的`panic`的风险
4. 正因为是我们来决定`channel`何时关闭， 所以我们避免了由于重复关闭`channel`而产生的`panic`的风险 
5. 在编译时使用了类型检查器，以防止对`channel`不正确的写入

其次我们来看看那些在从`channel`读取数据时可能发生的阻塞操作，作为`channel`的消费者，只需要关注两件事：

- 要知道一个`channel`什么时候被关闭
- 负责处理任何原因产生的`block`

处理第一点，我们可以像这样用第二个返回值`v,ok := <-channel`，来校验  
第二点就比较麻烦了，他取决于我们的算法逻辑，处理方法可能是作超时处理，也有可能别的程序告知要停止读取，或者也有可能就是像在程序生命周期内进行阻塞。不管是那种，作为一个消费端你应该处理读取过程中可能或将会阻塞的情况。  

现在呢，我们来通过一个例子来验证上面的观点，我们来创建一个`goroutine`,并且拥有一个`channel`，而且有一个消费端负责处理`channel`的阻塞`channel`关闭的情况：

```go
chanOwner := func() <-chan int {
  // 创建了一个5个buffer的channel,因为我们知道有6个数据将要写入，为了goroutine能尽快处理完
  resultStream := make(chan int, 5) 
  // 创建一个匿名函数来执行对resultStream的写操作
  go func() {
    // 我们确保写入数据完毕后关闭resultStream channel, 因为作为channel的owner这是我们应有的责任
    defer close(resultStream)
    for i:=0;i<=5;i++ {
      // 返回channel,暴露给外界，resultStream会隐式转换成单向channel
      resultStream <-i
    }
  }()
  return resultStream
}

resultStream := chanOwner()
// range over resultStream, 作为消费端，我们只关心阻塞和关闭的channel
for result := range resultStream {
  fmt.Printf("Received: %d\n", result)
}
fmt.Println("Done receiving!")
```

结果是：

```
Received: 0
Received: 1
Received: 2
Received: 3
Received: 4
Received: 5
Done receiving!
```

注意看下上面的`resultStream`这个`channel`是如何封装在`channel owner`函数里的。很明显，这种写法规避了往`nil channel`和`closed channel`里写入数据的风险，并且关闭`channel`只触发一次。为我们的程序规避了一大片的风险。  

而且这里建议你在程序里尽可能的保证`channel`的`物主身份`，也就是上面例子的`goroutine`的作用域范围小一点，这样好处是上面的一些问题会显而易见，如果你的`channel`是`struct`一个成员属性，并且有很多成员方法，那么随着代码的迭代很快就会不清楚`channel`的是来干什么的了  

作为`channel`的消费端，消费函数只有对`channel`的读取权限，因此也只需如何处理读取阻塞和`channel`的关闭。  
上面的小例子我们完美的阻塞的程序，直到`channel`被关闭  

如果你按照这个模式来coding, 会很容易像你期望的那样运行，而不会出现一些奇奇怪怪的情况。虽然不会100%保证不会出现问题，但是如果真的出现了问题，那么可能是你的`channel 物主身份`的作用域过大导致`channel`的职责混乱了。

## Select 

在各种并发模式里，`select`可以说随处可见，如果说`channel`是把一个个`goroutine`连接起来的桥梁，那么`select`是Go语言并发里最重要的语法之一。通过它把`channel`粘结在一起。  
`select`也可以安全的将`channel`的一些概念比如取消，超时，等待，默认值等组合在一起。  

我们来简单的过一边`select`用法:

```go
var c1, c2 <-chan insterface{}
var c3 chan<- interface 
select {
  case <- c1:
    // Do something
  case <- c2:
    // Do something
  case c3<- struct{}{}:
    // Do something
}
```

和`switch-case`语法很像，但是`select`并不是`case`从上到下顺序执行的，并且就算一条都没有满足条件，程序也不会继续往下执行后面的逻辑。  

我们知道在有数据或者已经关闭的`channel`情况下读取并且在还没达到`channel`容量上限的情况下写入，所有的`channel`的写入和读取我们认为是同时进行的，如果读取和写入都没有准备好，那么整个`select`就会阻塞，然后当其中一个`channel`准备好了，那么对应的`case`会被执行：  

```go
start := time.Now()
c := make(chan interface{})
go func() {
  time.Sleep(5*time.Second)
  close(c) // 等待5秒后关闭channel
}()

fmt.Println("Blocking on read...")
select {
  case <-c: // 这里我们试图从channel里读取数据
  fmt.Printf("Unblocked %v later .\n", time.Since(start))
}
```

结果输出：

```
Blocking on read...
Unblocked 5.000170042s later
```
从上面结果看我们在进入`select`代码块过了5秒后才结束。可以用这个方法简单而有效的等待一些工作处理好之前阻塞住。但是我们思考一下下面的情况会发生什么：  

- 如果有多个`channel`要读取会怎样？
- 如果永远不会有任何`channel`满足条件会怎样？
- 如果我想要执行一些操作，但是当时并没有`channel`满足条件该怎么办？  

第一个问题很有趣，多个`channel`会怎么分配呢？

```go
c1 := make(chan interface{});close(c1)
c2 := make(chan interface{});close(c2)

var c1Count, c2Count int
for i:=1000;i>=0;i-- {
  select {
    case <-c1:
      c1Count++
    case <-c2:
      c2Count++
  }
}
fmt.Printf("c1Count: %d\nc2Count: %d\n", c1Count, c2Count)
```

结果：

```
c1Count: 505
c2Count: 496
```

我们可以看到，1000次循环，基本每个`channel`触发次数各占一半。Go的`runtime`会运行一个伪随机，在`select`里均匀地选择`case`来触发，每个`case`都有同等的机会被选择。  

那么第二个问题呢：如果永远不会有任何`channel`满足条件会怎样？  
如果在阻塞期间你没有什么可做的，但是你又不想一直阻塞下去，那么你可以设置超时时间。Go里的`time`包提供了一个优雅的通过`channel`的方式来做这件事：

```go
var c <-chan int
select {
  case <-c: // 这里永远不会被执行到，因为我们是从一个nil channel 里读数据，会一直阻塞
  case <-time.After(1 * time.Second):
    fmt.Println("Timed out.")
}
```

`time.After` 这个方法会在参数给定的时间区间之后，会把当前时间发送到`channel`里，正好被`select`消费掉，达到了超时的作用。以后我们会基于这个理念来做一个更强壮的超时退出模式。

最后一个问题：如果我想要执行一些操作，但是当时并没有`channel`满足条件该怎么办？
这个就和`switch-case`差不多了，`select-case`也有个`default`默认执行条件：

```go
start := time.Now()
var c1, c2 <-chan int
select{
  case <-c1:
  case <-c2:
  default:
    fmt.Printf("In default after %v\n", time.Since(start))
}
```

结果：

```
In default after 1.421μs
```

可见几乎是瞬间就走到了`default`语句。这样我们就和`for-select`相互组合，来实现一个`goroutine`等待另一个`goroutine`的结果的同时继续处理其他事情：

```go
done := make(chan interface{})
go func() {
  time.Sleep(5*time.Second)
  close(done)
}()

workCounter := 0
loop:
for {
  select {
    case <-done:
      break loop
    default:
  }
  // 模拟处理一些其他事情
  workCounter++
  time.Sleep(1*time.Second) 
}
fmt.Printf("Achieved %v cycles of work before signalled to stop.\n", workCounter)
```

结果：

```
Achieved 5 cycles of work before signalled to stop.
```

最后, `select{}` 这个没有任何条件的语句会永远进行阻塞。


{{< alert icon="lightbulb">}}
To Be Continued...
{{< /alert >}}

  
## 本文参考

- [《Concurrency in GO》](https://raw.githubusercontent.com/KeKe-Li/book/master/Go/Concurrency in Go.pdf)
