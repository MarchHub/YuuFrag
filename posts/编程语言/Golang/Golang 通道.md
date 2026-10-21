---
tags:
  - Go
  - 并发编程
  - 异步与调度
---

# Golang 协程通道

## 总述

通道（Channel）用于协程之间的数据通信，通道操作自身可安全地并发使用，并为对应的发送和接收提供同步关系，但不会自动消除其他共享状态的数据竞态。通道本质上是一个队列（FIFO）

我们可以使用如下语法来声明一个通道
```go
var ch chan int
```
其中 `int` 可以换成其他不同的数据类型，甚至可以是空接口 `interface{}` 或者通道 `chan int`

接着实例化这个通道：
```go
ch = make(chan int)
```

以上两步可以化简成一步推导式
```go
ch := make(chan int)
```

以及对于单向的 channel ——

```go
// 只能发送
var sendCh chan<- int = ch

// 只能接收
var recvCh <-chan int = ch
```

## 通道数据操作

使用`<-`这个双目运算符表示数据的流动方向，可以用来往通道内写入数据`ch <- data`，也可以从通道中读取数据`temp = <- ch`

例子：
```go
func SendData(ch chan string) {
    defer close(ch)
    ch <- "first Data"
    ch <- "Second Data"
    ch <- "3rd Data"
}
func OutputData(ch chan string) {
    for data := range ch {
        fmt.Println(data)
    }
}
func main() {
    ch := make(chan string)
    go SendData(ch)
    OutputData(ch)
}
```

## 深入点

### 阻塞理解

channel 的阻塞，就好像是线下交易，一手交钱一手交货，必须双方同时到场准备好才可以进行交易。如果缺失任何一方，或是任何一方没有准备好，都无法进行这笔交易。

再比如说我们准备坐上8620蒸汽机车，
- 无缓：乘客（发送方）和售票员（接收方）必须同时到站台才能完成上车/买票
- 有缓：乘客可以先上车（缓冲区未满），或者售票员先坐着等乘客；只有车厢满了，才需要等有人下车

### 无缓

发送方`ch <- data` 会一直阻塞，直到接收方准备好

```go
func NoBufChannelSender(ch chan int) {
    fmt.Println("Data Preparing")
    ch <- 86
    fmt.Println("Data already been sent")
}
func ChannelDataReceiver(ch chan int) {
    time.Sleep(time.Second) // 假设复杂操作使当前线程停留 1s
    fmt.Println("准备接收")
    fmt.Println(<-ch)
    fmt.Println("接收结束")
}
func main() {
    ch := make(chan int)
    go NoBufChannelSender(ch)
    ChannelDataReceiver(ch)
}
```

执行结果（可能的一种，收发完成后的输出交错由调度决定）：
```text
Data Preparing
准备接收
86
接收结束
Data already been sent
```

执行顺序：
1. `Sender`输出准备数据，并被阻塞在`ch <- 86`
2. `Receiver`在`Sleep`后输出`准备接收`，并在`<-ch`准备就绪
3. 数据读取结束后，`Sender` 和 `Receiver` 都可以继续执行，具体顺序由调度决定
4. 所以 `Data already been sent` 可能出现在 `86` 之前，也可能出现在 `接收结束` 之后

所以可以看出
- 发送方 `ch <- data` 会阻塞，直到有接收方 `data := <-ch` 准备就绪。
- 接收方 `data := <-ch` 会阻塞，直到有发送方 `ch <- data` 发来数据。

### 有缓

- 发送方 `ch <- v` 只在缓冲区 **已满** 时阻塞，否则立刻入队。
- 接收方 `<-ch` 只在缓冲区 **为空** 时阻塞，否则立刻取队首。

```go
func main() {
    ch := make(chan string, 2)               // 创建一个缓存容量为 2 的通道
    fmt.Println("Main: 放入 A")
    ch <- "A"                                // 不阻塞
    fmt.Println("Main: 放入 B")
    ch <- "B"                                // 不阻塞
    go func() {
        time.Sleep(300 * time.Millisecond)
        fmt.Println("Goroutine: 接收", <-ch) // 取出 A，空出一个 slot
    }()
    
    fmt.Println("Main: 放入 C（会阻塞直到有接收）")
    ch <- "C"                                // 阻塞，直到上面 goroutine `<-ch` 发生
    fmt.Println("Main: C 已放入")
}
```

执行结果（可能的一种）：
```text
Main: 放入 A
Main: 放入 B
Main: 放入 C（会阻塞直到有接收）
Goroutine: 接收 A
Main: C 已放入
```

假如此时 `ch` 是一个无缓冲的通道，那么在往其中写入 `A` 数据的时候就已经会出现死锁了（因为当前协程一直阻塞在这里，没有一个数据的接收者）

### 非阻塞尝试与超时控制

可以使用 `select` 的 `default` 分支实现非阻塞尝试，或者使用 `time.After` 实现超时控制

## 注意

`chan` 作为参数传递的时候，虽然是值类型但是依旧是“传递语义”，类似于 map / slice，它们都是多个 view 视图去指向其底层实例，比如

```go
func produce(out chan<- int) {
    out <- 42
}

func consume(in <-chan int) {
    value := <-in
    fmt.Println(value)
}

func main() {
    ch := make(chan int)
    done := make(chan struct{})

    go produce(ch)

    go func() {
        consume(ch)
        close(done)
    }()

    <-done

    fmt.Println("main: 所有任务完成")
}
```

那么此时虽然 `ch` 作为值传递，但是生产者和消费者依旧操作的是同一个底层的 channel

这样就比较好理解在函数调用链中传递同一个 `context` 的时候，所有方法都可以检测其是否 `Done`（