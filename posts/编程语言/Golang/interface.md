---
tags:
  - Go
  - 类型系统
  - 面向对象
---

# interface

首先利用接口来作为一种 “能力”, 接着利用组合的方式来进行能力的 “叠加”, 这是非常经典的一套玩法, 比如在 C# 中, 我们可以通过多继承 interface

```C#
interface IReader
{
    void Read();
}

interface IWriter
{
    void Write();
}

// 通过多继承 interface 叠加能力
interface IReadWriter : IReader, IWriter
{
}

class File : IReadWriter
{
    public void Read()
    {
        Console.WriteLine("Read");
    }

    public void Write()
    {
        Console.WriteLine("Write");
    }
}
```

那么在 go 中, 使用隐式的实现接口来规避大量的继承操作 —— 有点像鸭子类型的感觉, “我只要实现了这个接口, 我就可以被声明作这个接口”, 即一个类型, 只要实现了一个声明接口中的所有方法, 它就可以作为这个接口类型进行使用, 而不需要显式的写出继承 / 组合关系, 比如 ——

```go
type Reader interface {
	Read()
}

type Writer interface {
	Write()
}

// 通过组合 interface 叠加能力
type ReadWriter interface {
	Reader
	Writer
}

type File struct{}

func (File) Read() {
	fmt.Println("Read")
}

func (File) Write() {
	fmt.Println("Write")
}

func main() {
	var r Reader = File{}
	var w Writer = File{}
	var rw ReadWriter = File{}

	r.Read()
	w.Write()

	rw.Read()
	rw.Write()
}
```

`File` 实现了 `Reader`、`Writer` 和 `ReadWriter`，因此 `File` 的值可以赋给这些接口类型的变量

不过值得注意的一点, 此处的类型检查是发生在编译期的, 不像 Python 之类的在运行期动态查询.

综上所述, go 查询的是一个类型所实现的方法 “集合”, 如果接口类型是其实现的方法子集, 则该类型可以作为这个接口来使用.

以及在 go 中有特殊例子 —— `interface{}`, 空接口, 即 `any` (从 1.18 版本启用的, 作为空接口的别名)

所有的类型都 “实现” 了空接口, 所以所有的类型都可以作为空接口来使用 —— 

```go
var a interface{} = 10
var b any = "hello world"
...
```

这样都是合法的, 不过值得注意的, 我们既然做了具体的静态类型隐藏, 声明其静态类型为 `any`, 就无法默认其拥有其原本类型所对应的行为, 也就是其现在的行为必须符合空接口的能力,那么想要恢复具体的类型, 就需要进行类型断言 ——

```go
var value any = 10

n, ok := value.(int)

if ok {
	fmt.Println(n + 1)
}
```

比如此处将其转化回 `int` 类型, 并且做一份值拷贝存放在 `n` 中以进行使用. 以及对于单返回值类型的类型断言, 在类型转化的时候类型不匹配无法转化, 则会 `panic`, 比如 ——

```go
// unsafe assertion
n := value.(int)
```

以及可以进行动态分发 ——

```go
func Handle(value any) {
	switch v := value.(type) {
	case int:
		fmt.Println("int:", v)

	case string:
		fmt.Println("string:", v)

	case bool:
		fmt.Println("bool:", v)

	default:
		fmt.Println("unknown type")
	}
}
```