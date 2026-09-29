# tcp-server

基于 `net` 的简单 TCP Server 实现

首先是创建监听

```go
const listenAddress = "127.0.0.1:4000"
listener, err := net.Listen("tcp", listenAddress)
if err != nil {
	return fmt.Errorf("listen on %s: %w", listenAddress, err)
}
defer listener.Close()
```

语义为创建一个 Socket 连接绑定在 `listenAddress` 上，并且进入 Listen 的监听状态。下面的异常处理和 `defer` 太典（

不过此处尚未有某一个具体的客户端进行连接，我们仅仅是在监听状态而已

```go
conn, err := listener.Accept()
if err != nil {
	return err
}
defer conn.Close()
```

接着创建一个连接，默认情况下 `.Accept()` 是阻塞等待，直到有客户端和当前的 Socket 建立连接。不过值得注意的，`listener` 负责接收连接，而 `conn` 表示一次具体的连接，所以之后的数据通信等，都是需要通过 `conn` 来进行

接着对于客户端来说，可以主动进行连接

```go
conn, err := net.Dial("tcp", "127.0.0.1:4000")
if err != nil {
	return err
}
defer conn.Close()
```

即通过 `.Dial` 对传入的目标端口进行连接，此时执行的时候系统大概率会自动分配临时端口，所以客户端不需要手动传入自己的端口。

此处执行`.Dial`的链路大体为 —— 创建 Socket，获得本机 IP 并且分配端口，执行 connect 操作，然后通过 TCP 三次握手稳定连接，让双方进入 `ESTABLISHED` 状态

那么在成功建立稳定连接之后，双方现在都拥有一个 `conn`，这是 golang 对于网络连接的一个抽象 ——

```go
Read([]byte) (int, error)
Write([]byte) (int, error)
Close() error

LocalAddr() net.Addr
RemoteAddr() net.Addr

SetDeadline(t time.Time) error
SetReadDeadline(t time.Time) error
SetWriteDeadline(t time.Time) error
```

不过实际上最主要聚焦的还是前三个操作

在建立连接之后 ——

```go
_, err = conn.Write([]byte("hello"))
if err != nil {
	return err
}
```

利用 `Write` 向 TCP 字节流中写数据，那些底层的分段/发送/ACK/排序等依旧是交付给内核实现，此处只是把 `hello` 这个字符串转成 `byte` 之后交付给 TCP 字节流而已

接着再看接收数据

```go
buf := make([]byte, 1024)

n, err := conn.Read(buf)
if err != nil {
	return err
}

fmt.Println(string(buf[:n]))
```

使用 `Read` 表示从 TCP 字节流中最多读取 `len(buf)` 的数据出来，其返回值 `n` 表示这次读取中字节流中实际读取了多少字节的数据，以及在已经完成连接的情况下，`Read` 方法在没有数据传输进来的时候也是默认阻塞的，直到收到数据，或者连接关闭或者到超时时间等才会继续执行

但是要小心 —— TCP 虽然提供了稳定有序的字节流，但是不能保证切割边界（字节流是这样的），读写次数不一致，所以在应用层需要自己做好数据的完整读取

最后关闭连接，`conn.Close()` 此时 TCP 会进入关闭连接的阶段，同时对于一端发送的数据，当另一端读取完整字节流之后，继续 `Read` 会读入一个 `io.EOF` 来作为结束，所以也可以有如下循环

```go
for {
	n, err := conn.Read(buf)

	if err == io.EOF {
		break
	}

	if err != nil {
		return err
	}

	handle(buf[:n])
}
```
