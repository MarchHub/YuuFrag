# Variant

定义一块内存空间，可以存储不止一种类型的变量 —— 从这个说法上有点像是 `Union`，不过行为上更加丰富

比较重要的，`Union` 只提供一块可以被多种类型复用的内存空间，但是在具体生产环节，对象还有析构/构造/复制/移动，甚至是异常处理等复杂的生命周期 / 内存管理问题，此时  `Variant` 就会帮忙进行管理（因为变量拥有所有权）

## 类型分发

有点丑陋，为什么不能引入下类似于 Rust 中的 `match-enum` 操作（~~看看隔壁 C#~~

在实际业务逻辑中，我们可能需要针对不同的类型分发不同的行为，比如客户端消息吧，传统的 OOP ——

```C++
class IMessage {
public:
    virtual ~IMessage() = default;
    virtual void Execute() = 0;
};

class MoveMessage : public IMessage {
	void Execute() override {
		...
	}
};

class RunMessage : public IMessage {
	void Execute() override {
		...
	}
};

...
```

然后在运行时简单的

```C++
IMessage* msg;
msg->Execute();
```

当然代价也很明显 —— 虚函数寻址间接执行函数带来的额外开销，而且接口对应的语义应该是“拥有这个能力”，而不是表达由于类型本身带来的行为差别 —— 这份行为是业务逻辑，而不是类型本身（这样说有点抽象，实际上就是对象本身不应该负责这份业务逻辑，它可以是纯数据+类型 tag，而真正的业务逻辑由于可能需要访问到许多外部方法，应当是在 Runtime 遇到这个类型的时候由业务逻辑的执行模块再来进行分发

那也可以利用模板元编程在编译期就做好类型分发（尤其是在 20 的标准下写起来更是方便），这样确乎运行时开销小了，而且也确乎是表达了“不同类型拥有不同行为”的语义 —— 但是问题在于参数本身必须在**编译期已知**，也就是对于在 Runtime 才知道类型的情况，就无法通过模板实现编译期类型分发

比如说现在类型标记使用 `tag` 实现，然后 `tag` 信息是来自 web 端接收的信息，`tag = 1` 表示事件 A；2 表示事件 B 等；那么我们需要手动在运行时来手写 `if-else` 进行分发 —— 因为具体的信息只能在运行时知道，在编译期编译器无法知道要生成什么类型对应的模板；并且其作为”值“，而不是类型参数；

与此同时，也可以使用函数重载实现

但是上述实现的前提都是调用点必须已经拥有一个具有具体静态类型的表达式，比如 ——

```C++
Move move;
Process(move);
```

即需要已知 Message 的静态类型才可以使用模板 / 重载实现

## 实现对齐

回到模式匹配的事情，Rust 给出了一份不错的玩法 ——

```rust
enum Message {
    Quit,                          // 单元变体 ≈ 携带 monostate
    Move { x: i32, y: i32 },       // 携带多个字段，支持解构
    Write(String),
}

fn handle(m: Message) {
    match m {
        Message::Quit            => println!("bye"),
        Message::Move { x, y }   => println!("move to ({x}, {y})"),
        Message::Write(s)        => println!("write {s}"),
    }   // 漏掉任何一个变体 → 编译错误（E0004），而不是运行期异常
}
```

重新认真阅读一下，其实这段代码包含了两件事情 —— 一个是所有的消息类型都是已知的（一个闭合的集合）；另一个是我们需要在运行时对于不同的消息类型做出不同的行为

简单的对齐，所以 `std::variant` 第一个比较重要的语义 —— 接下来存储的数据只可能是来自我声明的类型中的某一种，也就是常说的 Sum Type

然后再看 `match` 模块，此处其实干了两件事情 —— 一是检查类型，二是业务逻辑

那么在 C++ 中，我们可以朴素的实现，手动利用 `tag` 来进行类型检查

```c++
struct Message {
    enum class Tag { Quit, Move, Write } tag;
    int x = 0, y = 0;      // 只有 Move 用
    std::string s;         // 只有 Write 用
};

void handle(const Message& m) {
    switch (m.tag) {
    case Message::Tag::Quit:  std::cout << "bye\n"; break;
    case Message::Tag::Move:  std::cout << "move to (" << m.x << ", " << m.y << ")\n"; break;
    case Message::Tag::Write: std::cout << "write " << m.s << "\n"; break;
    }
}
```

这样的坏处在于空间浪费；那么好一点使用 `union`，但是就需要面对手动写构造/析构等的问题（`std::string` 不是一个善茬），并且每次修改 Tag 都需要修改对应的数据

不过上述的两种实现都存在共同的问题 —— Tag 和数据没有对应约束；`switch` 没有编译期检查是否有遗漏

现在把纯数据给拆出来，然后借助函数重载得到一份还可以的实现

```C++
struct Quit  {};
struct Move  { int x, y; };
struct Write { std::string s; };
using Message = std::variant<Quit, Move, Write>;

void process(const Quit&)     { std::cout << "bye\n"; }
void process(const Move&  mv) { std::cout << "move to (" << mv.x << ", " << mv.y << ")\n"; }
void process(const Write& w)  { std::cout << "write " << w.s << "\n"; }

void handle(const Message& m) {
    std::visit([](const auto& msg) { process(msg); }, m);
}
```

此处是借助 `std::visit` 把 `Message` 类型变量的真实的静态类型传给 visitor，也就是此处的 lambda 分发入口，然后再借助函数重载来执行具体的业务逻辑

那么也可以使用模板

```C++
template<class> inline constexpr bool always_false_v = false;

template<class T>
void process(const T& msg) {
    if constexpr (std::is_same_v<T, Quit>) {
        std::cout << "bye\n";
    } else if constexpr (std::is_same_v<T, Move>) {
        std::cout << "move to (" << msg.x << ", " << msg.y << ")\n";
    } else if constexpr (std::is_same_v<T, Write>) {
        std::cout << "write " << msg.s << "\n";
    } else {
        static_assert(always_false_v<T>, "未处理的 Message 变体");  // 穷尽性兜底
    }
}

void handle(const Message& m) {
    std::visit([](const auto& msg) { process(msg); }, m);
}
```

此处借用了`std::visit` 会要求 visitor 对 `variant` 中所有可能的 alternative 都是合法可调用的这一特性，使得 `process` 会生成所有合法分支对应的实现

接着对于当前这个例子，由于业务逻辑非常薄，所以也可以 ——

```C++
// 利用继承把多个 Lambda 的 operator() 合并 成一个可调用对象
template<class... Ts>
struct overloaded : Ts...
{
    using Ts::operator()...;
};

// C++20 起可省略
template<class... Ts>
overloaded(Ts...) -> overloaded<Ts...>;

std::visit(
    overloaded{
        [](const Quit&)
        {
            std::cout << "bye\n";
        },

        [](const Move& msg)
        {
            std::cout << "move "
                      << msg.x << ", "
                      << msg.y << '\n';
        },

        [](const Write& msg)
        {
            std::cout << msg.s << '\n';
        }
    },
    message
);
```

上述写法其实有点像是

```C++
struct MessageHandler {
    void operator()(const Quit&)  const { ... }
    void operator()(const Move&)  const { ... }
    void operator()(const Write&) const { ... }
};

void handle(const Message& m) {
    std::visit(MessageHandler{}, m);
}
```

这样的好处在于 `MessageHandler` 可以更方便的保存状态，比如 Log 之类的实现。不过单纯从语法上说，其实更像是

```C++
struct QuitHandler  { void operator()(const Quit&)  const { std::cout << "bye\n"; } };

struct MoveHandler  {
    void operator()(const Move& mv) const {
        auto [x, y] = mv;
        std::cout << "move to (" << x << ", " << y << ")\n";
    }
};

struct WriteHandler { void operator()(const Write& w) const { std::cout << "write " << w.s << "\n"; } };

struct MessageHandler : QuitHandler, MoveHandler, WriteHandler {
    using QuitHandler::operator(), MoveHandler::operator(), WriteHandler::operator();
};
```

反正都挺神秘的，如果需要同时对多个 `std::variant` 做类型分发就更是依托 —— 因为为了保证所有分支合法，我们需要实现 $m \times n$ 个分发逻辑，这没什么问题，但是心智负担上 —— 不同的分发大概率有重叠逻辑， 怎么写比较优雅？对于新加入的分支如何添加更少的分发逻辑？

总的来说就是如何偷懒 —— 感觉实际上可以混用模板和重载实现（~~不过有这种需求感觉也挺神秘的~~）—— 借助模板来实现公共逻辑，然后再借助重载来实现特定逻辑，不赖

再做点小补充 —— 其实不使用 `visit`，也可以手动读取 `index` 或者使用 `holds_alternative` 做行为分发

## 真的 Variant

上面太”激进“的在说明模式匹配这件事情，实际上 `variant` 很干净 —— 现在我有一个变量，可能存储的是下面几种类型中的某一种，我现在可以在运行时切换存储的类型，同时可以知道我现在存的是什么类型

在切换的时候，会结束之前存储的资源的生命周期（执行析构）然后构造出新的数据并且存储在原地

我们可以在运行时知道类型，比如 ——

```C++
// 仅查看类型
if (std::holds_alternative<Move>(m))
{
    // 当前是 Move
}

// 查看类型并且安全的获得对象
if (auto* move = std::get_if<Move>(&m))
{
    std::cout << move->x << ", " << move->y;
}
```

当然，如果非常明确对象类型，也可以 ——

```C++
auto& move = std::get<Move>(m);
```

不过如果此时 `m` 不是 `Move` 类型就会抛 `std::bad_variant_access` 异常

也可以通过 `m.index()` 获得当前类型的序号

所以简单来看其实也就是一个单纯的 Variable + Tag 的实现，同时好处在于生命周期管理以及绑定了类型和其 Tag