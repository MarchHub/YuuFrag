# C++ 编译流程

借助简单例子说明清楚 C++ 的编译到运行中的程序的过程，首先简单来说，我们会经过如下步骤 ——

```
.cpp -> 预处理 -> 编译 -> 汇编 -> .obj -> 链接 -> .exe
.exe -> Loader -> C/C++ 初始化 ->main
```

先定义一下当前的操作例子，然后再逐个步骤说明 ——

```C++
// math.hpp
#pragma once
int add(int a, int b);

// math.cpp
int add(int a, int b) { return a + b; }

// main.cpp
#include "math.hpp"
int main() { return add(114, 514); }
```

(以及接下来使用 `clang++` 作为例子，并且在 win 上运行)

## 独立编译

在编译过程中，值得注意的，它是**独立编译** —— 不是把整个项目一次性全部塞给编译器，而是通过一个个翻译单元进行编译，所以通常每一个 `.cpp` 都会经过独立的预处理、编译和汇编过程。

也就是说原则上每个 `.cpp` 都会作为一个独立的编译单元然后生成一个对应的 `.obj`，这样可以规避仅修改一两个文件但是依旧需要重新编译整个项目的问题。

## 预处理

预处理主要执行预处理指令（又是废话），比如 `#define` `#include` 之类，比如 `#include` 主要在预处理阶段的行为大体是把被 include 的文本复制到当前编译的文件中

比如可以观察到 ——

```bash
clang++ -E main.cpp -o main.i
```

接着打开 `main.i`可以看到 `#include` 语句已经不在，取而代之的是

```C++
int add(int a, int b);
# 2 "main.cpp" 2

int main() { return add(114, 514); }
```

此处的 `#...` 是预处理器留下的标记，可以让后续编译器依旧可以知道代码原本属于哪个文件的哪行。

类似的，宏定义做文本替换 / 删除注释等，也都是在这一步执行

## 编译

预处理之后的翻译单元，可以经过编译成为汇编。在预处理阶段主要处理的是文本层面上的事情，但是并没有涉及词法/语法分析等。而编译过程就会开始进行词法语法语义分析，模板实例化，AST，IR 生成，优化等管线，最终得到对应目标机器的汇编代码。

例如

```C++
int x = "Hello World!"
```

这类型的错误在预处理阶段就不会被检查出来，但是可以在编译阶段被发现。

我们可以查看一下编译结果 ——

```bash
clang++ -S main.cpp -o main.s
```

可以看到 `main.cpp` 中对于 `add` 方法的调用已经变成了冷冰冰的

```asm
movl    $114, %ecx
movl    $514, %edx                      # imm = 0x202
callq   "?add@@YAHHH@Z"
```

以及抽象语法树也可以进行查看

```bash
clang++ -Xclang -ast-dump -fsyntax-only main.cpp
```

查看 IR 中间表示 ——

```bash
clang++ -S -emit-llvm main.cpp -o main.ll
```

这两个步骤埋钩子，在之后具体深入说明编译原理的时候展开叙述

## 汇编

众所周知，汇编依旧是给人阅读的，还需要转成机器码给机器执行，于是此处需要借助汇编器将汇编转化成机器指令并且写入 `.obj` 文件 ——

```bash
clang++ -c main.s -o main.obj
clang++ -c math.s -o math.obj
```

需要分别对 `main.cpp` 和 `math.cpp` 执行一整套编译流程直到生成 Object File 为止，当然，`.obj` 包含了目标 CPU 可以使用的机器代码，但是它仍旧不是一个完整的可执行程序，只是一个结构化的二进制文件而已，我们可以查看 ——

```bash
llvm-objdump -h main.obj
```

得到

```
main.obj:       file format coff-x86-64
Sections:
Idx Name          Size     VMA              Type
  0 .text         00000021 0000000000000000 TEXT
  1 .data         00000000 0000000000000000 DATA
  2 .bss          00000000 0000000000000000 BSS
  3 .xdata        00000008 0000000000000000 DATA
  4 .debug$S      00000098 0000000000000000 DATA, DEBUG
  5 .pdata        0000000c 0000000000000000 DATA
  6 .llvm_addrsig 00000001 0000000000000000
```

可以看出，实际上我们的代码是会被分割成若干代码段的。继续查看

```bash
llvm-nm main.obj
llvm-nm math.obj
```

对应的，可以察觉在 `main.obj` 中只有 `add` 这个方法的未定义引用（` U ?add@@YAHHH@Z` 会查看到这种东西，是 Name Mangling 后的结果），而 `math.obj` 中是 `add` 方法的实现。所以非常顺其自然的引出了链接器的事情 —— 将调用函数的文件和实现函数的文件进行对应

这也是得到虽然得到了一个 `.obj` 但是依旧无法执行的原因 —— `U` 表示 Undefined，我们需要 `call` 它，但是其实并不知道它具体在哪，所以只能把此处调用的地址先留空，等待之后 Linker 进行目标地址的“补全”

## 链接

现在我们已经分别有了 `main.obj` 和 `math.obj`，现在仅需要通过链接器然后得到最终的可执行文件即可 ——

```bash
clang++ main.obj math.obj -o app.exe
```

链接器会先 Object Files 中的各种 Symbol 进行解析（常见的 `undefined symbol` 错误就是出现在这个阶段）；随后决定各个 Section 在最终可执行文件中的布局，并根据 Relocation 信息修正代码和数据中的地址引用，最终生成 `.exe`

## 执行

接下来简单说明一下让冰冷的 `*.exe` 如何在 CPU 上运行。首先在 Win 下，`*.exe` 通常使用 Portable Executable 格式，也就是一种结构化的二进制文件，它不仅包含可执行的部分本身，还记录了许多操作系统在加载程序的时候所需要的信息，比如程序有哪些 Section，它们如何映射到虚拟地址空间，程序依赖那些 DLL，程序执行入口等

当程序被启动时，Windows 会首先创建对应的进程和初始线程，并建立程序运行所需要的基本虚拟地址空间。随后 Loader 会根据 PE 文件中的信息，将程序映射到当前进程的虚拟地址空间中，以及完成一些在加载阶段必须要完成的工作，比如映射需要的 DLL —— 假设当前程序使用到了某 DLL，但是编译和链接的时候并不一定可以知道这个 DLL 在运行时会被放进哪个虚拟地址中，所以需要在此时进行动态链接，使得当前程序可以找到 DLL 中的函数。

PE 文件记录了程序执行入口 ——

```bash
llvm-readobj --file-headers app.exe
```

可以借此查找到 `AddressOfEntryPoint` 这样的字段，记录程序进入的执行入口。

不过程序执行入口和 `main` 函数又不是对等的关系 —— 需要先进行程序运行前必须完成的一些任务，初始化 C/C++ Runtime 所需要的运行时环境，在 C 的语境下就是 CRT ( C Runtime Library )，C++ 可能会依赖此，并且添加一些新东西。等这一切执行之后，才会进入 `main` 函数的执行。