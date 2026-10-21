---
tags:
  - C++
  - 并发编程
  - 同步机制
---

# Atomic

`std::atomic`, 用于定义原子变量, 允许多个线程按照 C++ 内存模型进行读写等操作. 一般用于线程之间共享状态等.

## 基础语法

```C++
std::atomic<int> count{1};

int x = count.load();          // x = 1
count.store(114);
x = count.fetch_add(514);      // x 仍是 114
```

## 基本语义

可以查下[CPP Reference](https://en.cppreference.com/cpp/atomic/atomic), 里面提到 ——

- **Store**: Writes
- **Load**: Reads
- **Read-modify-write (RMW)**: Reads and then writes. Note: This is one atomic operation, not several.

有这三种基础语义, 那么简单来说 —— 写数据 / 读数据 / 读旧数据然后进行修改最后完整写入数据到原子变量中

再简单化简 API 的话, 通过 `.load()` 进行原子读取; `.store()` 进行原子写入, 然后 RMW 的操作 —— 比如 `fetch_add`: `int old = count.fetch_add(1);` 先读取旧数据到变量中, 然后做加法再写入数据

类似的, 也提供了

- fetch_sub()
- fetch_or()
- fetch_and()
- fetch_xor()

类似操作

与此同时,还有一个 RMW 比较常用的操作是 —— `.exchange`, 语义是先读取旧数据,然后再写入, 比如——

```C++
bool old = busy.exchange(true);
```

不过值得注意的是, 如果拆成两个原子操作, 并不能保证和起来是一个原子操作(就是别拆成先`load`再`store`)

## CAS

大名鼎鼎的 `Compare-and-Swap`, 直白的翻译下 —— ”比较数据,然后在符合条件的时候交换数据“, 不过还是归属在 RMW 中的操作, 提供了两个常用 API ——

- compare_exchange_weak
- compare_exchange_strong

例子 ——

```C++
std::atomic<int> state{0};

int expected = 0;
int write = 1;

bool success = state.compare_exchange_strong(
	expected,
	write);
```

表达我认为当前 state 的值是 epected, 如果确乎如此的话, 我希望写入 write 的值到 state 中. 然后返回一个是否写入成功的 bool 判断

不过这里有一个注意点 ——

```C++
std::atomic<int> state{ 0 };
int expected = 114;
bool success = state.compare_exchange_strong(expected, 1);

std::cout << "Success: " << success << std::endl;
std::cout << "State: " << state.load() << std::endl;
std::cout << "expected: " << expected << std::endl;
```

我们会发现运行结果为

```
Success: 0
State: 0
expected: 0
```

也就是说, `expected != state` 虽然写入数据失败了, 但是 `expected` 依旧会被修改.

不过其实从它提供的函数签名也可以看出一些端倪, 要求传入的第一个参数是一个可变的左值引用.

接着看`weak`和`strong`的区别 ——

`weak` 允许 “虽然 expected 的值和原子变量值相等,但是依旧可能出现修改原子变量数据失败的情况”, 也就是我们常说的 "spurious failure" (伪失败, 比如底层的原子更新尝试失败, 像是其他核心访问了相关缓存状态/上下文切换, 比较典型 —— Load-Linked / Store-Conditional 架构上的 reservation 丢失)

而 `strong` 在契约上保证了不因为 spurious failure 而返回失败.

所以常见的,可以写 ——

```C++
do {
    desired = f(expected);
} while (!counter.compare_exchange_weak(
	    expected,
	    desired));

if (state.compare_exchange_strong(expected, newData)) {
	...
}
```

主要看调用方是否允许 spurious failure 的存在, 比如单次的 if 可能就表达某种比较强的业务状态 —— 有就是行, 没有就是不行, 于是使用 strong;

而与此同时, Lock-Free 中常见的 Loop 使用 weak 进行构造 —— 不断尝试当前值直到 counter 成为目标值, 此时哪怕出现了 spurious failure 也问题不大,继续尝试就是.

## 内存顺序

前面的 API 可以保证单次操作是原子的, 但是并没有保证在多个线程之间的执行顺序 —— 比如我们希望 A 是生产者, 可以给变量赋值为 1145, 然后让 B 作为消费者去读取.

接着我们使用了原子变量来进行存储 —— 线程 A 执行了 store, 原子地写入,没问题; B 执行了 load, 原子地读取, 没问题. 但是我们依旧不知道到底是 B Load 的是 A 的 Store 还是其他什么

于是就需要引入一个可见的先后顺序(happens-before)来定义这件事情. —— 只有 A happens before B, 则说明 B 发生的时候绝对不能和 A 还没有发生的效果是一致的. ( 好绕

在单个线程内可能涉及指令重排, 在多线程时可能出现数据竞态, 我们都可以使用 happens-before 这样的模型去让结果确定. 不过此处主要说明在多线程环境下针对原子操作提供的一套机制

于此同时, happens before 还有传递性(废话), 如果 A happens before B, 同时 B happens before C, 那么 A happens before C. 所以利用传递性, 假设我们可以借助单线程内的语句 “sequenced-before” 然后借助原子变量跨线程的 "synchronizes-with", 来实现利用原子变量来保护其他资源的效果.

当然, 对于一个 Atomic 对象, 还有其内部的`modification order`, 说明了对其所有的修改的顺序是如何.

C++ 中定义了

```C++
enum class memory_order {
    relaxed,
    acquire,
    release,
    acq_rel,
    seq_cst
};
```

我们查看[草案](https://eel.is/c%2B%2Bdraft/atomics#order),里面说到 ——

> The enumeration memory_order specifies the detailed regular (non-atomic) memory synchronization order as defined in [intro.multithread](https://eel.is/c%2B%2Bdraft/intro.multithread "6.10.2)Multi-threaded executions and data races" and may provide for operation ordering

也就是说, 这个参数说明了当前这次的原子操作在 C++ memory model 中承担了什么 顺序 / 同步 语义

不过值得说明的, 这个并不是什么 CPU 执行顺序之类的神秘东西, 或许可以算是一种数学建模? 先说个小例子

```
Thread 1
A: data = 42
B: ready.store(true, release)

Thread 2
C: ready.load(acquire)
D: read data
```

单个线程的 sequence-before 可以保证 `A -> B`, `C -> D`, 那么此时 C 在 Load 的时候, 有可能是读取到了 B 写入的数据, 也有可能是其他什么(比如初始值之类的), 那么当 C 读取到了 true 的时候, 我们才说 `A -> B -> C -> D` 这一整条同步链成立

也就是说我们应当关注的是操作达成的“效果” ( 比如操作 A 的效果 B 是否可见/观测 ), 而不是说真的形成某种执行顺序

### Relaxed

非常的放松 a, 只要当前的操作是原子性的即可,不需要为别的对象建立 happens before 之类的语义. 所以非常典, 只需要按照自己作为 Atomic Object 的 modification order 即可

```C++
std::atomic<std::uint64_t> requests{0};

void OnRequest()
{
    requests.fetch_add(
        1,
        std::memory_order_relaxed);
}
```

比如此处的语义就是, 线程 A 调用就 +1; B 调用就再 +1; 只要保证自己的 +1 是原子操作即可.

### Release / Acquire

- `Release` 表示让一次 Store / RMW 成为一条同步链的输入端(输入数据, 相当于一次发布数据
- `Acquire` 表示让一次的 Load / RMW 成为一条同步链的输出端(输出数据, 可以接收 Release 发布的数据

重点是如何让一条同步链连起来 —— 也非常人性化, 对于一个 Atomic 对象, 事件 A 是 Release 事件, 事件 B 是 Acquire 事件, 然后 A, B 对同一个 Atomic 对象操作, 并且保证了 B 读取到 A 的值, 那么事件 A 就 happens before B

还是例子 ——

```C++
data = std::atomic<int>{0};
data.store(114514, std::memory_order_release);
auto x = data.load(std::memory_order_acquire);
```

假设 x **读取成功**, 返回 114514, 那么非常好, 可以证明数据就是 `data.store` 存进去的. 此时就构造了 `data.store -> data.load` 这样一条同步链

当然, 此时依旧会受到 modification order 的影响, 比如 ——

```C++
Thread A:
state.store(1, release)

Thread C:
state.store(2, relaxed)

Thread B:
state.load(acquire)
```

我们依旧无法确定, B 读取的是 A 还是 C 的数据.

### acq_rel

我们知道有 RMW 操作 —— 它可能是读取旧值然后进行修改写入数据, 那么此时它应当既可以作为一条同步链的输入端, 也可以作为另一条同步链的输出端.

`acq_rel` 就是解决这样一件事情, (其实看命名就非常好懂了) —— 被声明这个顺序的原子操作, 可以是上一条同步链的 `acquire` (输出端), 同时写入数据可以作为下一条同步链的 `release` (输入端)

那么常见的操作 ——

- exchange
- compare_exchange
- fetch_add (fetch 一整个操作族)

### seq_cst

唔, 这又是另一个故事 —— 它的意思是在多个原子操作之间在符合各自的`acquire`（获取）和 `release`（释放）的所有同步语义的前提之下, 额外保证了**所有** `seq_cst` 操作之间存在一个单一的全局先后顺序

先看个简单例子, 假设我们现在有两个操作, 分别是资源初始化,和前台服务监听这个资源, 那么可以建立两个原子变量来进行存储状态

```C++
std::atomic<bool> resources_ready{false};
std::atomic<bool> service_running{false};
```

然后写初始化的线程

```C++
void Initialize()
{
    LoadResources();

    resources_ready.store(
        true,
        std::memory_order_seq_cst);

    StartService();

    service_running.store(
        true,
        std::memory_order_seq_cst);
}
```

再写监控线程

```C++
void Monitor()
{
    if (service_running.load(std::memory_order_seq_cst))
    {
        bool ready = 
	        resources_ready.load(std::memory_order_seq_cst);

        if (!ready)
        {
            // 这在当前设计中意味着状态出现矛盾
            ReportInvalidState();
        }
    }
}
```

首先再确定一下我们的预想 —— 服务在资源完成加载之后才开始运行

我们先定义一下

```C++
Initialize:
	resources_ready.store(true, std::memory_order_seq_cst); // A
	service_running.store(true, std::memory_order_seq_cst); // B

Monitor:
	service_running.load(std::memory_order_seq_cst)         // C
	resources_ready.load(std::memory_order_seq_cst);        // D
```

也就是说 C++ 抽象机 要推导出一个同一条全局逻辑顺序可以让被声明 `seq_cst` 的操作合理运行.

首先初始化线程本身要求了 `A -> B` ( sequenced-before ), 同理 `Monitor` 要求了 `C -> D`

然后在 C 读取到 True 的时候, 说明`A -> B` 已经发生在 `C -> D` 之前了, 那么此时 D 就不可能再访问  的旧数据, 所以 `ReportInvalidState` 不会被触发

当然, 在此处这个例子其实太麻烦了, 不太恰当, 一个变量足以, 或者原子变量的类型是一个状态枚举也不错.

那我们再举一个抽象例子 ——

```C++
std::atomic<bool> x{false};
std::atomic<bool> y{false};

bool r1 = false;
bool r2 = false;

Thread 1:
	x.store(true, std::memory_order_seq_cst); // A
	r1 = y.load(std::memory_order_seq_cst);   // B
Thread 2:
	y.store(true, std::memory_order_seq_cst); // C
	r2 = x.load(std::memory_order_seq_cst);   // D
```

那么是否存在一个非初始状态是 `r1 == false && r2 == false` ?

显然不可能, 先由于 sequence-before 要求 `A -> B`, `C -> D`, 那么当 `r1 == false` 的时候, 说明 B 执行在 C 前面 (因为没有 load 到 C 中的数据), 也就是 `B -> C`, 此时又因为 `r2 == false`, 说明 D 执行在 A 前面, 也就是 `D -> A`,  最后再利用一下传递性可以得到  `A -> B -> C -> D -> A` 这形成了一个环, 所以不可能是一个可以执行的操作序列. 于是绝对不存在这样的一个非初始状态.

odl, 值得注意的是, 如果留空, 默认就是 `seq_cst`

## 综上所述

原子操作首先保证单次访问不可分割; 而多个线程之间还需要考虑操作之间的**排序、可见性以及同步关系**

`memory_order` 用于描述一次 atomic 操作承担的内存排序语义，并由此参与构造 `synchronizes-with`、`happens-before` 等关系
