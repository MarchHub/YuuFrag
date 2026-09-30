---
title: map
tags:
  - Go
  - 类型系统
date: 2025-08-13
---
# map

哈希表，平均 `O(1)` 查询，在 Golang 中值得注意 —— **未同步的并发读写不安全**

## 底层原理

不是基于红黑树的实现；从 Go 1.24 开始，内置 map 使用基于 Swiss Table 的哈希表实现——

每个 group 包含 8 个键值槽位和一个 8 字节的 control word，用来快速比较哈希值的部分位，减少对比完整 key 的次数。

如果发生哈希冲突，会通过开放寻址在后续 group 中探测可用槽位，而不是指向旧实现中的 `overflow bucket`。

如果 table 增长到容量上限，会通过替换或拆分 table 完成扩容

## 基础使用

```go
table := make(map[string]int)
table["Tom"] = 114

val, ok := table["Jack"]
fmt.Println(val, ok)

delete(table, "Tom")

for k, v := range table {
    fmt.Println(k, v)
}
```

遍历的时候，`k-v` 的顺序未指定，也不保证与前一次遍历相同

## Set

由于 golang 没有提供 `set[T]`，所以通常使用 `map[T]struct{}` 来进行使用

```go
visited := make(map[string]struct{})
visited["Alice"] = struct{}{}

_, ok := visited["Alice"]  // 查询
```

比较好的，`struct{}` 是零大小类型，相当于只是一个 flag，说明我们现在仅关心 Key 是否存在

不过实际上已经有比较成熟的库实现 [golang-set](https://github.com/deckarep/golang-set)

## 注意

小坑点吧

```go
type Point struct {
    X int
    Y int
}

positions := map[string]Point{}

positions["ID_1"].X = 1  // 非法
```

`map` 元素不可直接寻址，所以需要

```go
p := positions["ID_1"]
p.X = 1
positions["ID_1"] = p
```

才是对于 Value 为结构体的时候的合法修改，其原因也比较简单，`map` 返回的是值，而不是一个拥有稳定地址的对象，这样的好处在于，`map` 自己在扩容等的时候，旧地址失效，那么缓存的变量就不会出现垂悬指针之类的麻烦问题

不过也有另外一种常见的解法，即 Value 类型不再是 `T`，而是 `*T`， 即存储一个指针，这样其哪怕返回值，依旧可以直接原地修改

```go
players := map[string]*Player{
    "ID_1": {},
}
players["ID_1"].X = 1
```