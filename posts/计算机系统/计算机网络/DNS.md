---
tags:
  - 网络协议
  - 计算机网络
---

# DNS

通过分层递归的方式，实现域名到IP的映射

首先域名是按层级划分的，比如 `www.example.com` —— 可以按 `.` 来划分其层级，顶级域名为 `com`，接着是 `example` 最后是 `www`（不过其实有一个 `root`，由于大家都一样所以经常被省略

我们可以使用 `dig` 来进行查看 ——

```bash
machillka@SEXY-MACHINE:~$ dig www.example.com

; <<>> DiG 9.18.39-0ubuntu0.24.04.7-Ubuntu <<>> www.example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34913
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;www.example.com.               IN      A

;; ANSWER SECTION:
www.example.com.        245     IN      A       172.66.147.243
www.example.com.        245     IN      A       104.20.23.154

;; Query time: 7 msec
;; SERVER: 10.255.255.254#53(10.255.255.254) (UDP)
;; WHEN: Mon Sep 28 20:05:22 CST 2026
;; MSG SIZE  rcvd: 76
```

首先

```bash
;; QUESTION SECTION:
;www.example.com.               IN      A
```

表示当前是 “ 对`www.example.com`进行 IPV4 地址的查询 ”，最后得到结果为

```bash
www.example.com.        245     IN      A       172.66.147.243
www.example.com.        245     IN      A       104.20.23.154
```

说明当前这个域名对应两个 IPV4 地址，其中 245 表示一个 TTL （Time 2 Live，存活时间），说明当前这份记录还可以存活 245 秒

对于当前的例子，首先先对于 Root 进行一次查询，如果 Root 说明要委托给下一个 Name Server 的地址，然后委托下一个 Name Server 进行查询，比如此处是对于 `.com.` 进行询问，接着以此类推，`.com.` 的 Name Server 一般也不会直接保存这个域名对应的 IP，而是告诉 Resolver `example.com.` 由那些 Name Server 管理，然后委托给 `example.com.` 的 Name Server 来进行查询……

不过值得注意的是，并不是每一个`.`都会进行一次分层查询的委托，而是“有委托的时候才进行委托”，比如对于`a.b.c.d.e.f.g.com`来说，就大概率不会对每一个字母进行进行委托

如果想要查看查询过程，可以使用 `+trace` 参数进行查看
