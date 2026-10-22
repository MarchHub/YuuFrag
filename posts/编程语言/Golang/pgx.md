# pgx

记录一下借助 `pgx` 对 `PostgreSQL` 的使用

首先考虑对于一个数据库，考虑一次请求如何保证事务的 ACID；对于多次请求，如何缩短用户得到回复的时间？

## 连接池

假设使用 TCP 连接一个数据库，那么一次 `ping-pong` 至少需要经过 TCP 的握手，可能存在的 TLS 层，数据库认证层等多个层级，也就是说一次连接其实是很贵的操作。那么对于每一次 SQL 查询，如果都针对一次连接，那么延迟将会非常高

于是针对此问题，又是一个经典的"开池“的玩法，类似于对象池，提前准备好多个连接存入池中，然后每一次事务请求只需要从池中找一个空闲的连接即可

对应在 `pgx` 中，也有独立的连接类型 `pgx.Conn` 和 `pgxpool.Pool` 两个类型；使用 `Conn` 的话语义为建立一条独立的连接，并且程序需要自己管理其生命周期（安全关闭），所以确乎是可以直接使用 `Conn` 来实现连接池的效果 —— 但值得注意的，其并非**并发安全**，即同一条 `Conn` 不允许多个 `goroutine` 同时执行数据库操作。不太方便蝶苏哇

不过依旧提供一个基础例子

```go
ctx := context.Background()

conn, err := pgx.Connect(
    ctx,
    os.Getenv("DATABASE_URL"),
)
if err != nil {
    return err
}
defer conn.Close(ctx)

var value int
err = conn.QueryRow(
    ctx,
    "SELECT 42",
).Scan(&value)
```

于是使用连接池，方便许多 —— `Pool` 的实现是并发安全的，所以大部分情况下可以放心的直接调用 `pool.Query`, `QueryRow` 等 API，不过也可以通过 `Acquire` 来从池中借用一条连接，并且通过 `Release` 将使用权归还给连接池。那么在 `Acquire` 的情况下，使用权暂时移交给了 Pool 外部，那么 Pool 逻辑上可以视作有一条 Conn 被占用；但是使用权在外部的 Conn 就不是 Pool 来实现的并发安全，所以如果有 ——

```go
conn, _ := pool.Acquire(ctx)
defer conn.Release()

go conn.Exec(ctx, "SELECT 1")
go conn.Exec(ctx, "SELECT 2")
```

那么无法保证其并发安全（

## 事务

一条事务本身的 ACID 是数据库保证的，比如在数据库中，可以利用 `BEGIN`, `COMMIT` 来保证事务一致性

在 golang 实现中，通常使用 `tx, err := pool.Begin()` 来从连接池中获取一个空闲连接并且声明事务的开始，归还该连接的时机是 `tx.Rollback` 或 `tx.Commit` 两个方法执行之后，其中在 `Commit` 执行之后，如果再执行 `Rollback` 会因为 `dbTx` 已经关闭而返回 `ErrTxClosed`，使得事务不会回滚（详情可以参考一下[源码实现](https://github.com/jackc/pgx/blob/master/tx.go#L247)

先给出一份常见的写法

```go
tx, err := pool.Begin(ctx)
if err != nil {
    return err
}
defer tx.Rollback(context.Background())

_, err = tx.Exec(ctx, "UPDATE ...")
if err != nil {
    return err
}

_, err = tx.Exec(ctx, "UPDATE ...")
if err != nil {
    return err
}

return tx.Commit(ctx)
```

梳理逻辑，先是利用 `Begin` 获取连接并且声明事务开始，然后借助 `defer Rollback` 来保证如果没有正常执行到 `Commit` 则会触发 `Rollback` 回滚事务，避免函数中途因为错误返回而遗留未结束的事务。中间是正常的执行数据库逻辑，如果途中遇到某 `err` 也可以进行额外的业务逻辑异常的操作，不赖，粒度更细一些