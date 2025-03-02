# 
volatile 关键字

## 用户:
cv (const and volatile) type qualifiers
 C++ C++ language Declarations 
Appear in any type specifier, including decl-specifier-seq of declaration grammar, to specify constness or volatility of the object being declared or of the type being named.

const - defines that the type is constant.
volatile - defines that the type is volatile.
Explanation
Any (possibly incomplete) type other than function type or reference type is a type in a group of the following four distinct but related types:

A cv-unqualified version.
A const-qualified version.
A volatile-qualified version.
A const-volatile-qualified version.
These four types in the same group have the same representation and alignment requirements.

Array types are considered to have the same cv-qualification as their element types.

const and volatile objects
When an object is first created, the cv-qualifiers used (which could be part of decl-specifier-seq or part of a declarator in a declaration, or part of type-id in a new-expression) determine the constness or volatility of the object, as follows:

A const object is
an object whose type is const-qualified, or
a non-mutable subobject of a const object.
Such object cannot be modified: attempt to do so directly is a compile-time error, and attempt to do so indirectly (e.g., by modifying the const object through a reference or pointer to non-const type) results in undefined behavior.
A volatile object is
an object whose type is volatile-qualified,
a subobject of a volatile object, or
a mutable subobject of a const-volatile object.
Every access (read or write operation, member function call, etc.) made through a glvalue expression of volatile-qualified type is treated as a visible side-effect for the purposes of optimization (that is, within a single thread of execution, volatile accesses cannot be optimized out or reordered with another visible side effect that is sequenced-before or sequenced-after the volatile access. This makes volatile objects suitable for communication with a signal handler, but not with another thread of execution, see std::memory_order). Any attempt to access a volatile object through a glvalue of non-volatile type (e.g. through a reference or pointer to non-volatile type) results in undefined behavior.
A const volatile object is
an object whose type is const-volatile-qualified,
a non-mutable subobject of a const volatile object,
a const subobject of a volatile object, or
a non-mutable volatile subobject of a const object.
Behaves as both a const object and as a volatile object.
Each cv-qualifier (const and volatile) can appear at most once in any cv-qualifier sequence. For example, const const and volatile const volatile are not valid cv-qualifier sequences.


mutable specifier
mutable - permits modification of the class member declared mutable even if the containing object is declared const (i.e., the class member is mutable).
May appear in the declaration of a non-static class members of non-reference non-const type:

class X
{
    mutable const int* p; // OK
    mutable int* const q; // ill-formed
    mutable int&       r; // ill-formed
};
mutable is used to specify that the member does not affect the externally visible state of the class (as often used for mutexes, memo caches, lazy evaluation, and access instrumentation).

class ThreadsafeCounter
{
    mutable std::mutex m; // The "M&M rule": mutable and mutex go together
    int data = 0;
public:
    int get() const
    {
        std::lock_guard<std::mutex> lk(m);
        return data;
    }
 
    void inc()
    {
        std::lock_guard<std::mutex> lk(m);
        ++data;
    }
};
Conversions
There is partial ordering of cv-qualifiers by the order of increasing restrictions. The type can be said more or less cv-qualified than:

unqualified < const
unqualified < volatile
unqualified < const volatile
const < const volatile
volatile < const volatile
References and pointers to cv-qualified types can be implicitly converted to references and pointers to more cv-qualified types, see qualification conversions for details.

To convert a reference or a pointer to a cv-qualified type to a reference or pointer to a less cv-qualified type, const_cast must be used.

Notes
The const qualifier used on a declaration of a non-local non-volatile non-template(since C++14)non-inline(since C++17) variable that is not declared extern gives it internal linkage. This is different from C where const file scope variables have external linkage.

The C++ language grammar treats mutable as a storage-class-specifier, rather than a type qualifier, but it does not affect storage class or linkage.

Some uses of volatile are deprecated:

lvalue of volatile type as operand of built-in increment/decrement operators;
lvalue of volatile type as left operand of built-in direct assignment, unless the direct assignment expression appears in an unevaluated context or is a discarded-value expression;
volatile object type as function parameter type or return type;
volatile qualifier in structured binding declaration.
(since C++20)
Keywords
const, volatile, mutable

Example
Run this code
#include <cstdlib>
 
int main()
{
    int n1 = 0;          // non-const object
    const int n2 = 0;    // const object
    int const n3 = 0;    // const object (same as n2)
    volatile int n4 = 0; // volatile object
 
    const struct
    {
        int n1;
        mutable int n2;
    } x = {0, 0};        // const object with mutable member
 
    n1 = 1;   // OK: modifiable object
//  n2 = 2;   // error: non-modifiable object
    n4 = 3;   // OK: treated as a side-effect
//  x.n1 = 4; // error: member of a const object is const
    x.n2 = 4; // OK: mutable member of a const object isn't const
 
    const int& r1 = n1; // reference to const bound to non-const object
//  r1 = 2; // error: attempt to modify through reference to const
    const_cast<int&>(r1) = 2; // OK: modifies non-const object n1
 
    const int& r2 = n2; // reference to const bound to const object
//  r2 = 2; // error: attempt to modify through reference to const
//  const_cast<int&>(r2) = 2; // undefined behavior: attempt to modify const object n2
 
    [](...){}(n3, n4, x, r2); // see also: [[maybe_unused]]
 
    std::system("g++ -O3 -Wa,-adhln ./main.cpp"); // may issue asm on POSIX systems
}
Possible output:

# typical machine code produced on an x86_64 platform
# (only the code that contributes to observable side-effects is emitted)
main:
    movl    $0, -4(%rsp) # volatile int n4 = 0;
    movl    $3, -4(%rsp) # n4 = 3;
    xorl    %eax, %eax   # return 0 (implicit)
    ret
这是cppreference中关于volatile关键词的表述
 我没看明白 你能仔细帮我讲解一下吗?

## ChatGPT:
当然，很乐意为您详细讲解 C++ 中的 `volatile` 关键字。您提供的 cppreference 文档是关于 `volatile` 的权威解释，我将基于此文档，并结合代码示例，力求让您彻底理解。

**`volatile` 关键字的核心作用：告诉编译器“这个变量的值可能会在程序控制之外被改变，不要对它进行优化”。**

这句话是理解 `volatile` 的关键。让我们逐步展开：

**1. `volatile` 是一个类型修饰符 (type qualifier)**

就像 `const` 一样，`volatile` 是用来修饰类型的。它可以和 `const` 一起使用，也可以单独使用。

```c++
volatile int v_var; // 声明一个 volatile 的 int 变量
const volatile int cv_var = 10; // 声明一个 const volatile 的 int 变量
```

**2. `volatile` 的目的：处理外部变化**

`volatile` 主要用于以下几种场景，这些场景的共同特点是变量的值可能会在程序代码“不知情”的情况下发生改变：

* **硬件寄存器 (Hardware Registers):**  在嵌入式系统或驱动程序开发中，某些内存地址可能对应硬件寄存器。这些寄存器的值可能被硬件设备随时修改，而不是由程序代码显式写入。例如，读取传感器数据、控制外围设备等。
* **内存映射 I/O (Memory-Mapped I/O):**  类似于硬件寄存器，某些内存区域可能被映射到外部设备，设备可以直接修改这些内存区域的值。
* **多线程环境 (Multithreading - 早期用法，现在不推荐):**  在早期的 C++ 中，`volatile` 曾被误用作线程间同步的手段。但现代 C++ 提供了更强大、更安全的工具（如 `std::atomic` 和 `std::mutex`）来处理多线程并发问题，`volatile` **不应该** 用于线程间通信。  **文档中也明确指出，`volatile` 不适合用于线程间通信，应该使用 `std::memory_order`。**
* **信号处理函数 (Signal Handlers):**  信号处理函数是异步执行的，可能会在程序执行的任何时刻被调用。如果信号处理函数需要访问某些全局变量，这些变量应该声明为 `volatile`，以确保编译器不会对这些变量的访问进行过度优化。

**3. `volatile` 如何阻止优化？**

编译器为了提高程序性能，会进行各种优化，例如：

* **寄存器优化:**  编译器可能会将变量的值存储在寄存器中，而不是每次都从内存中读取。如果变量被频繁访问，寄存器优化可以显著提高速度。
* **代码重排序:**  编译器可能会为了指令流水线效率，对代码的执行顺序进行调整，只要不改变程序的最终结果。
* **死代码消除:**  如果编译器认为某些代码段的结果不会被用到，可能会直接删除这些代码。

但是，对于 `volatile` 变量，编译器必须 **非常保守** 地处理：

* **每次访问 `volatile` 变量，都必须直接从内存中读取或写入。**  编译器不能将 `volatile` 变量的值缓存到寄存器中，也不能假设变量的值在两次访问之间保持不变。
* **`volatile` 变量的读写操作不能被重排序。**  `volatile` 访问必须按照代码中出现的顺序执行，不能与其他可见的副作用（side-effect）操作重排序。
* **对 `volatile` 变量的访问不能被优化掉。**  即使编译器认为某个 `volatile` 变量的读取结果没有被使用，它仍然必须执行读取操作，因为读取操作本身可能具有外部可见的副作用（例如，读取硬件寄存器可能会触发某些硬件动作）。

**总结 `volatile` 的关键特性：**

* **不优化:**  阻止编译器进行可能导致程序行为不符合预期的优化。
* **强制内存访问:**  每次读写都直接操作内存，而不是寄存器缓存。
* **顺序保证:**  `volatile` 操作的顺序必须与代码顺序一致。
* **副作用可见:**  每次 `volatile` 访问都被视为可见的副作用。

**4. `const volatile`**

一个变量可以同时是 `const` 和 `volatile`。这意味着：

* **`const` 部分:**  程序代码 **不能** 通过赋值等方式修改这个变量的值。
* **`volatile` 部分:**  这个变量的值 **可能** 在程序控制之外被修改。

`const volatile` 常用于表示一个内存区域，其值可能被硬件或其他外部因素修改，但程序本身不应该去修改它，只能读取它的值。例如，表示一个只读的硬件状态寄存器。

**5. `mutable` (与 `volatile` 无直接关系，但常被提及)**

`mutable` 是用于类成员变量的修饰符。它允许在 `const` 成员函数中修改 `mutable` 成员变量。

```c++
class MyClass {
public:
    mutable int count;
    int value;

    void incrementCount() const {
        count++; // OK: mutable 成员可以在 const 成员函数中修改
        // value++; // 错误: 非 mutable 成员不能在 const 成员函数中修改
    }
};
```

`mutable` 的目的是允许修改对象的某些内部状态，而仍然保持对象在外部观察上的常量性。它通常用于实现缓存、互斥锁等内部机制，这些机制的改变不应该影响对象的逻辑 constness。

**`mutable` 和 `volatile` 的区别：**

* `mutable` 关注的是 **类成员变量的修改权限**，允许在 `const` 对象或 `const` 成员函数中修改特定的成员。
* `volatile` 关注的是 **变量值的外部可变性**，强制编译器每次都从内存中读取变量的值，并阻止优化。

它们是不同的概念，但有时会在某些场景下一起出现，例如，一个 `const volatile` 对象可能包含 `mutable` 成员，但这并不常见。

**6.  被弃用的 `volatile` 用法 (C++20 起)**

C++20 标准开始弃用 `volatile` 的一些用法，主要是因为这些用法在现代 C++ 中有更好的替代方案，并且容易被误用：

* **`volatile` 左值作为自增/自减运算符的操作数:**  例如 `++v_var;` 或 `--v_var;`  (不推荐)
* **`volatile` 左值作为赋值运算符的左操作数:** 例如 `v_var = 10;` (除非在未求值上下文或丢弃值表达式中) (不推荐)
* **`volatile` 对象类型作为函数参数或返回值类型:**  例如 `void func(volatile int v);` 或 `volatile int func();` (不推荐)
* **结构化绑定声明中的 `volatile` 修饰符:**  (不推荐)

这些被弃用的用法主要是因为它们在多线程编程中容易产生误导，并且现代 C++ 提供了更明确、更强大的工具（如 `std::atomic`）来处理并发和内存模型问题。

**7.  代码示例解析**

让我们仔细分析您提供的代码示例：

```c++
#include <cstdlib>

int main()
{
    int n1 = 0;          // non-const object
    const int n2 = 0;    // const object
    int const n3 = 0;    // const object (same as n2)
    volatile int n4 = 0; // volatile object

    const struct
    {
        int n1;
        mutable int n2;
    } x = {0, 0};        // const object with mutable member

    n1 = 1;   // OK: modifiable object
//  n2 = 2;   // error: non-modifiable object
    n4 = 3;   // OK: treated as a side-effect
//  x.n1 = 4; // error: member of a const object is const
    x.n2 = 4; // OK: mutable member of a const object isn't const

    const int& r1 = n1; // reference to const bound to non-const object
//  r1 = 2; // error: attempt to modify through reference to const
    const_cast<int&>(r1) = 2; // OK: modifies non-const object n1

    const int& r2 = n2; // reference to const bound to const object
//  r2 = 2; // error: attempt to modify through reference to const
//  const_cast<int&>(r2) = 2; // undefined behavior: attempt to modify const object n2

    [](...){}(n3, n4, x, r2); // see also: [[maybe_unused]]

    std::system("g++ -O3 -Wa,-adhln ./main.cpp"); // may issue asm on POSIX systems
}
```

* **`volatile int n4 = 0;` 和 `n4 = 3;`**:  这两行代码演示了 `volatile` 的基本用法。即使编译器开启了优化 (`-O3`)，它也必须生成代码来执行 `n4 = 0;` 和 `n4 = 3;` 这两次写入操作。这是因为 `n4` 是 `volatile` 的，编译器不能假设这两次写入是冗余的，或者可以被优化掉。每次对 `n4` 的赋值都被视为一个重要的副作用。

* **汇编输出 (Possible output):**  示例中提供的汇编代码片段：

```assembly
main:
    movl    $0, -4(%rsp) # volatile int n4 = 0;
    movl    $3, -4(%rsp) # n4 = 3;
    xorl    %eax, %eax   # return 0 (implicit)
    ret
```

可以看到，即使在 `-O3` 优化级别下，编译器仍然生成了将 `0` 和 `3` 写入内存地址 `-4(%rsp)` 的指令。`-4(%rsp)` 很可能就是变量 `n4` 在栈上的地址。  这证明了 `volatile` 阻止了编译器优化，确保了对 `n4` 的写入操作被实际执行。

* **其他部分 (关于 `const`, `mutable`, `const_cast`):**  代码示例的其他部分主要演示了 `const` 和 `mutable` 的用法，以及 `const_cast` 的使用和潜在风险。这些部分与 `volatile` 的关系不大，主要是为了展示 C++ 中类型修饰符和类型转换的一些特性。

**总结:**

`volatile` 关键字在 C++ 中是一个相对底层的工具，主要用于处理与硬件或外部环境交互的场景。它强制编译器保守地处理变量的访问，阻止优化，确保每次读写都直接操作内存，并保证操作顺序。  在现代 C++ 中，`volatile` 的使用场景相对受限，尤其是在多线程编程中，应该优先考虑使用 `std::atomic` 等更专业的工具。  理解 `volatile` 的核心作用是理解它如何影响编译器的优化行为，以及它在处理外部变化时的必要性。

希望这个详细的解释能够帮助您理解 `volatile` 关键字。如果您还有任何疑问，请随时提出！

> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
文档中应该还有提到包含volatile元素的数组(容器)的情况吧
在这种情况下 能得到volatile那些优化阻止保证的 是那些元素而非容器本身吧(如果容器无这些修饰符的话)

## ChatGPT:
是的，您说得很对！文档中确实会提到包含 `volatile` 元素的数组和容器的情况。在这种情况下，`volatile` 关键字提供的优化阻止和保证是应用于 **元素** 而不是容器本身（如果容器本身没有 `volatile` 修饰符）。

让我们更详细地解释一下：

**1. `volatile` 数组**

当您声明一个 `volatile` 数组时，`volatile` 修饰符实际上是应用于数组的 **元素类型**，而不是数组变量本身。

例如：

```c++
volatile int volatile_array[10];
```

在这个声明中，`volatile` 修饰的是 `int` 类型。这意味着 `volatile_array` 是一个包含 10 个 `volatile int` 元素的数组。

**效果：**

* **每个元素都是 `volatile` 的：**  对 `volatile_array` 数组中 **每个元素** 的访问（读取或写入）都将被视为 `volatile` 访问。编译器不会对这些元素的访问进行优化，每次访问都会直接从内存中读取或写入。
* **数组本身不是 `volatile` 的：**  数组变量 `volatile_array` 本身并没有被声明为 `volatile`。这意味着，例如，如果您将 `volatile_array` 赋值给另一个数组变量，或者获取 `volatile_array` 的地址，这些操作本身不是 `volatile` 操作。

**示例代码：**

```c++
#include <iostream>

int main() {
    volatile int volatile_array[2];
    volatile_array[0] = 10; // volatile 写操作
    volatile_array[1] = 20; // volatile 写操作

    int val1 = volatile_array[0]; // volatile 读操作
    int val2 = volatile_array[1]; // volatile 读操作

    std::cout << "val1: " << val1 << ", val2: " << val2 << std::endl;

    return 0;
}
```

在这个例子中，每次访问 `volatile_array[0]` 和 `volatile_array[1]` 都会被视为 `volatile` 访问，即使在优化级别较高的编译下，编译器也会确保每次都直接访问内存。

**2. 包含 `volatile` 元素的容器**

对于标准库容器（如 `std::vector`, `std::list`, `std::array` 等），情况类似。当您声明一个容器来存储 `volatile` 类型的元素时，`volatile` 修饰符应用于容器的 **元素类型**。

例如：

```c++
std::vector<volatile int> volatile_vector;
```

这里，`std::vector` 存储的是 `volatile int` 类型的元素。

**效果：**

* **容器中的元素是 `volatile` 的：**  当您访问 `volatile_vector` 中的元素时（例如，通过 `operator[]`, `at()`, 迭代器等），您访问的是 `volatile int` 对象。因此，对这些元素的访问将是 `volatile` 访问，具有 `volatile` 关键字的所有特性（阻止优化、强制内存访问、顺序保证等）。
* **容器本身不是 `volatile` 的：**  `volatile_vector` 容器对象本身不是 `volatile` 的。您可以像操作普通 `std::vector` 一样操作 `volatile_vector`，例如添加元素、删除元素、获取大小等。这些容器操作本身不是 `volatile` 操作，除非您将容器对象本身也声明为 `volatile` (这种情况比较少见，通常不需要)。

**示例代码：**

```c++
#include <iostream>
#include <vector>

int main() {
    std::vector<volatile int> volatile_vector;
    volatile_vector.push_back(10); // 添加 volatile int
    volatile_vector.push_back(20); // 添加 volatile int

    volatile int& ref1 = volatile_vector[0]; // 获取 volatile int 的引用
    volatile int val2 = volatile_vector[1];  // volatile 读操作

    std::cout << "ref1: " << ref1 << ", val2: " << val2 << std::endl;

    ref1 = 30; // volatile 写操作 (通过引用修改)
    std::cout << "ref1 after modification: " << ref1 << std::endl;

    return 0;
}
```

在这个例子中，`volatile_vector` 存储的是 `volatile int` 元素。通过 `volatile_vector[0]` 获取的引用 `ref1` 也是 `volatile int&` 类型，因此对 `ref1` 的读写操作都是 `volatile` 操作。

**总结：**

当 `volatile` 与数组或容器一起使用时，关键在于 `volatile` 修饰的是 **元素类型**。这意味着 `volatile` 提供的保证和优化阻止是应用于容器或数组中存储的 **每个元素**，而不是容器或数组对象本身。

您的理解是完全正确的！ `volatile` 的作用域是元素级别，而不是容器级别（除非容器本身也被声明为 `volatile`，但这通常不是常见的用法）。

> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
SSE寄存器是什么？

## ChatGPT:
SSE 寄存器是 **Streaming SIMD Extensions (流式单指令多数据扩展)** 指令集使用的寄存器。为了理解 SSE 寄存器，我们需要先了解一下 SSE 指令集和 SIMD 的概念。

**1. SIMD (Single Instruction, Multiple Data) - 单指令多数据**

SIMD 是一种并行计算技术。它的核心思想是：**用一条指令，同时处理多个数据。**

传统的 CPU 指令通常是 SISD (Single Instruction, Single Data) - 单指令单数据。 也就是说，一条指令只能操作一对数据。例如，要将两个数组 `A` 和 `B` 的对应元素相加，传统的 SISD 方式需要循环遍历数组，每次循环执行一条加法指令，处理一对元素。

而 SIMD 技术允许一条指令同时操作多个数据对。例如，一条 SIMD 加法指令可以同时将两个包含多个元素的向量相加。这样可以显著提高数据并行任务的执行效率，尤其是在处理大量数据时，例如图像处理、音频处理、科学计算等。

**2. SSE (Streaming SIMD Extensions) - 流式 SIMD 扩展**

SSE 是一系列 Intel 和 AMD 处理器中引入的 SIMD 指令集扩展。它扩展了 x86 指令集，增加了新的指令和 **SSE 寄存器**，以支持 SIMD 操作。

SSE 经历了多个版本，例如 SSE, SSE2, SSE3, SSSE3, SSE4.1, SSE4.2 等，每个版本都增加了新的指令和功能。

**3. SSE 寄存器 (XMM, YMM, ZMM 寄存器)**

SSE 指令集引入了专门用于 SIMD 操作的寄存器。最初，SSE 引入的是 **XMM 寄存器**。 随着指令集的不断扩展，又引入了 **YMM 寄存器** 和 **ZMM 寄存器**。

* **XMM 寄存器 (128-bit):**  最初的 SSE 寄存器，每个寄存器可以存储 128 位的数据。 这可以容纳：
    * 4 个单精度浮点数 (float, 32-bit each)
    * 2 个双精度浮点数 (double, 64-bit each)
    * 16 个字节 (byte, 8-bit each)
    * 8 个字 (word, 16-bit each)
    * 4 个双字 (dword, 32-bit each)
    * 2 个四字 (qword, 64-bit each)

* **YMM 寄存器 (256-bit):**  随着 **AVX (Advanced Vector Extensions)** 指令集的引入，出现了 YMM 寄存器。 YMM 寄存器是 XMM 寄存器的扩展，每个寄存器可以存储 256 位的数据。 它可以容纳 XMM 寄存器两倍的数据量，例如：
    * 8 个单精度浮点数
    * 4 个双精度浮点数

* **ZMM 寄存器 (512-bit):**  随着 **AVX-512** 指令集的引入，出现了 ZMM 寄存器。 ZMM 寄存器是 YMM 寄存器的进一步扩展，每个寄存器可以存储 512 位的数据。 它可以容纳 YMM 寄存器两倍的数据量，例如：
    * 16 个单精度浮点数
    * 8 个双精度浮点数

**寄存器的数量:**

最初的 SSE 指令集只提供了 8 个 XMM 寄存器 (在 x86-64 架构中扩展到 16 个)。 随着 AVX 和 AVX-512 的引入，YMM 和 ZMM 寄存器的数量也增加到了 16 个 (在 x86-64 架构中，ZMM0-ZMM15，YMM0-YMM15, XMM0-XMM15)。  在某些更新的处理器和操作系统中，寄存器数量甚至可以更多 (例如，AVX-512 可以支持 32 个 ZMM 寄存器)。

**4. SSE 寄存器的用途**

SSE 寄存器主要用于存储和操作 **打包数据 (packed data)**，也就是将多个相同类型的数据元素打包到一个寄存器中。  SSE 指令可以对整个寄存器中的打包数据进行并行操作。

例如，使用 SSE 指令可以将两个 XMM 寄存器中的 4 对单精度浮点数同时相加，结果存储到另一个 XMM 寄存器中。

**5. SSE 寄存器的数据类型**

SSE 寄存器主要用于处理以下数据类型：

* **单精度浮点数 (float):**  用于科学计算、图形处理等。
* **双精度浮点数 (double):**  用于需要更高精度的科学计算。
* **整数 (Integer):**  SSE 指令集也支持对打包整数进行操作，例如字节、字、双字等。 这在多媒体处理、图像处理等方面非常有用。

**6. SSE 寄存器的优势**

使用 SSE 寄存器和 SSE 指令集的主要优势在于：

* **性能提升:**  通过 SIMD 并行处理，可以显著提高数据并行任务的执行速度。 特别是在处理大量数据时，性能提升非常明显。
* **能效提升:**  用一条 SIMD 指令完成多个操作，相比于多条 SISD 指令，可以减少指令的解码和执行开销，从而提高能效。

**7. 示例 (概念性)**

假设我们要将两个包含 4 个单精度浮点数的数组 `A` 和 `B` 相加，并将结果存储到数组 `C` 中。

**SISD (传统方式):**

```c++
for (int i = 0; i < 4; ++i) {
    C[i] = A[i] + B[i]; // 循环 4 次，每次执行一次加法指令
}
```

**SIMD (使用 SSE 寄存器和指令):**

1. **将数组 `A` 的 4 个元素加载到一个 XMM 寄存器 (例如 `xmm0`)。**
2. **将数组 `B` 的 4 个元素加载到另一个 XMM 寄存器 (例如 `xmm1`)。**
3. **使用 SSE 加法指令 (例如 `addps`) 将 `xmm0` 和 `xmm1` 相加，结果存储到另一个 XMM 寄存器 (例如 `xmm2`)。**  **一条指令完成 4 个加法操作！**
4. **将 `xmm2` 寄存器中的结果存储到数组 `C` 中。**

**总结:**

SSE 寄存器是 SSE 指令集的核心组成部分，它们是用于存储和操作打包数据的特殊寄存器，支持 SIMD 并行计算。 通过使用 SSE 寄存器和 SSE 指令，可以显著提高在多媒体处理、科学计算、图像处理等领域中数据并行任务的性能和能效。 随着技术发展，SSE 寄存器从 XMM 扩展到 YMM 和 ZMM，寄存器宽度和数量不断增加，为更强大的并行计算提供了硬件基础。

> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
嗯 这是一段简单的求和函数对应在O2优化下的汇编代码(因为我要刻意去看看XMM寄存器的用途)
请帮我非常详细地解析这段汇编代码。

## ChatGPT:
当然，我很乐意为您详细解析这段汇编代码，逐行解释其功能和原理，并重点关注 XMM 寄存器的使用。

**C++ 源代码回顾:**

```c++
#include <array>

int foo(const std::array<int, 4>& input)
{
    auto sum = 0xD; // sum 初始化为 13 (0xD in hexadecimal)
    for (auto element : input)
    {
        sum += element; // 循环累加数组元素
    }
    return sum; // 返回总和
}
```

**汇编代码 (x86-64, gcc 14.2, -O2 优化):**

```assembly
foo(std::array<int, 4ul> const&):
  movdqu  xmm0, XMMWORD PTR [rdi]  ; Line 2
  movdqa  xmm1, xmm0             ; Line 3
  psrldq  xmm1, 8                ; Line 4
  paddd   xmm0, xmm1             ; Line 5
  movdqa  xmm1, xmm0             ; Line 6
  psrldq  xmm1, 4                ; Line 7
  paddd   xmm0, xmm1             ; Line 8
  movd    eax, xmm0              ; Line 9
  add     eax, 13                ; Line 10
  ret                             ; Line 11
```

**逐行解析:**

**Line 1: `foo(std::array<int, 4ul> const&):`**

* 这是一个标签 (label)，表示函数 `foo` 的开始。
* `std::array<int, 4ul> const&`  表明函数 `foo` 接受一个常量引用 (`const&`) 指向 `std::array<int, 4>` 类型的参数。
* 在 x86-64 调用约定中，第一个参数通常通过 `rdi` 寄存器传递。 因此，`rdi` 寄存器将包含指向 `input` 数组首地址的指针。

**Line 2: `movdqu  xmm0, XMMWORD PTR [rdi]`**

* **`movdqu`**:  这是 SSE 指令 `Move Unaligned Double Quadword Unpacked` 的缩写。
    * **`mov`**:  移动数据指令。
    * **`d`**:  double quadword (128 bits)。
    * **`qu`**:  unaligned (未对齐的内存访问)。  `u` 表示允许从**未对齐**的内存地址读取数据。 如果数据确定是 16 字节对齐的，可以使用 `movdqa` (aligned move)，但 `movdqu` 更通用，可以处理未对齐的情况，虽然在某些情况下可能会稍微慢一些。
* **`xmm0`**:  目标寄存器，即 SSE XMM 寄存器 0。
* **`XMMWORD PTR [rdi]`**:  源操作数，表示从内存地址 `[rdi]` 处读取一个 128 位 (16 字节) 的数据。
    * **`XMMWORD PTR`**:  类型修饰符，显式指定内存中的数据被视为 128 位 (XMM word)。
    * **`[rdi]`**:  表示内存地址，`rdi` 寄存器中存储的是指向 `input` 数组首地址的指针。

**功能:**  这条指令将 `input` 数组 (4 个 `int` 元素，共 16 字节) 从内存加载到 `xmm0` 寄存器中。  由于 `std::array<int, 4>` 在内存中是连续存储的，所以一次 `movdqu` 指令就可以将整个数组加载到 `xmm0` 寄存器中。  `xmm0` 寄存器现在包含了 4 个 32 位整数，以 "packed" (打包) 的形式存储。 假设数组 `input` 的元素是 `[a, b, c, d]`，那么 `xmm0` 寄存器的内容 (概念上) 可以表示为 `[d, c, b, a]` (字节序可能影响实际存储顺序，但逻辑上是打包在一起的四个整数)。

**Line 3: `movdqa  xmm1, xmm0`**

* **`movdqa`**:  这是 SSE 指令 `Move Aligned Double Quadword Aligned` 的缩写。
    * **`mov`**: 移动数据指令。
    * **`d`**: double quadword (128 bits)。
    * **`qa`**: aligned (对齐的内存访问)。 `a` 表示要求内存地址必须是 16 字节对齐的。 在寄存器到寄存器之间的移动中，对齐通常不是问题，`movdqa` 在这种情况下通常与 `movdqu` 性能相近。
* **`xmm1`**:  目标寄存器，即 SSE XMM 寄存器 1。
* **`xmm0`**:  源寄存器，即 SSE XMM 寄存器 0。

**功能:**  这条指令将 `xmm0` 寄存器的内容 **复制** 到 `xmm1` 寄存器。  现在 `xmm0` 和 `xmm1` 都包含了数组 `input` 的数据。 复制的目的是为了后续的操作，避免修改原始数据。

**Line 4: `psrldq  xmm1, 8`**

* **`psrldq`**:  这是 SSE2 指令 `Packed Shift Right Logical Double Quadword` 的缩写。
    * **`psrl`**:  Packed Shift Right Logical (打包逻辑右移)。
    * **`dq`**:  double quadword (128 bits)。
* **`xmm1`**:  目标和源寄存器，即 SSE XMM 寄存器 1。  指令的结果会写回 `xmm1`。
* **`8`**:  立即数，表示右移的位数，单位是 **字节**。  8 字节 = 64 位。

**功能:**  这条指令将 `xmm1` 寄存器中的 128 位数据 **逻辑右移 8 个字节 (64 位)**。  逻辑右移会在左侧填充 0。  由于我们处理的是 32 位整数，右移 8 字节相当于将 `xmm1` 中的高 2 个整数部分移动到低 2 个整数部分的位置，并将高 2 个整数部分清零。

假设 `xmm1` 初始内容 (概念上) 是 `[d, c, b, a]`，执行 `psrldq xmm1, 8` 后，`xmm1` 的内容会变成 `[0, 0, d, c]`。  （注意字节序和实际存储顺序，这里只是概念表示）。

**Line 5: `paddd   xmm0, xmm1`**

* **`paddd`**:  这是 SSE2 指令 `Packed Add Doubleword` 的缩写。
    * **`padd`**:  Packed Add (打包加法)。
    * **`d`**:  doubleword (32 bits)。
* **`xmm0`**:  目标和源寄存器，即 SSE XMM 寄存器 0。 指令的结果会写回 `xmm0`。
* **`xmm1`**:  源寄存器，即 SSE XMM 寄存器 1。

**功能:**  这条指令执行 **打包的 32 位整数加法**。 它将 `xmm1` 寄存器中的 **打包的四个 32 位整数** 与 `xmm0` 寄存器中对应的 **打包的四个 32 位整数** 相加，并将结果存储回 `xmm0` 寄存器中。

结合之前的操作：
* `xmm0` 初始包含 `[d, c, b, a]`
* `xmm1` 现在包含 `[0, 0, d, c]`

执行 `paddd xmm0, xmm1` 后，`xmm0` 的内容会变成：
`[d+0, c+0, b+d, a+c]`  也就是 `[d, c, b+d, a+c]`。  **注意：这里实际上是将数组的相邻元素两两相加了。**

**Line 6: `movdqa  xmm1, xmm0`**

* **`movdqa  xmm1, xmm0`**:  与 Line 3 相同，再次将 `xmm0` 的内容复制到 `xmm1`。  目的是为了进行下一轮的移位和加法操作。

**Line 7: `psrldq  xmm1, 4`**

* **`psrldq  xmm1, 4`**:  与 Line 4 类似，但这次是逻辑右移 **4 个字节 (32 位)**。

假设 `xmm1` 初始内容 (从 Line 6 复制而来) 是 `[d, c, b+d, a+c]`，执行 `psrldq xmm1, 4` 后，`xmm1` 的内容会变成 `[0, d, c, b+d]`。

**Line 8: `paddd   xmm0, xmm1`**

* **`paddd   xmm0, xmm1`**:  再次执行打包的 32 位整数加法。

结合之前的操作：
* `xmm0` 现在包含 `[d, c, b+d, a+c]` (从 Line 5 结果)
* `xmm1` 现在包含 `[0, d, c, b+d]` (从 Line 7 结果)

执行 `paddd xmm0, xmm1` 后，`xmm0` 的内容会变成：
`[d+0, c+d, (b+d)+c, (a+c)+(b+d)]`  也就是 `[d, c+d, b+d+c, a+b+c+d]`。

**观察:** 经过这两轮移位和加法操作，`xmm0` 寄存器的 **最低位 (最右边) 的 32 位整数**  `a+b+c+d`  实际上已经包含了数组 `input` 所有元素的总和！  其他位置的整数是中间计算结果，但我们最终只需要总和。

**Line 9: `movd    eax, xmm0`**

* **`movd`**:  这是 SSE 指令 `Move Doubleword` 的缩写。
    * **`mov`**: 移动数据指令。
    * **`d`**: doubleword (32 bits)。
* **`eax`**:  目标寄存器，32 位通用寄存器 `eax` (它是 64 位寄存器 `rax` 的低 32 位部分)。
* **`xmm0`**:  源寄存器，即 SSE XMM 寄存器 0。

**功能:**  这条指令将 `xmm0` 寄存器 **最低位的 32 位数据** (一个 doubleword) 移动到 `eax` 寄存器中。  由于我们之前分析 `xmm0` 的最低位已经包含了数组元素的总和 `a+b+c+d`，所以这条指令实际上是将计算出的数组元素总和提取到了 `eax` 寄存器中。

**Line 10: `add     eax, 13`**

* **`add`**:  加法指令。
* **`eax`**:  目标和源寄存器，32 位通用寄存器 `eax`。
* **`13`**:  立即数，十进制数 13 (十六进制 0xD)。

**功能:**  这条指令将立即数 `13` (十进制) 加到 `eax` 寄存器中的值上，并将结果存储回 `eax` 寄存器。  回顾 C++ 代码，`sum` 变量被初始化为 `0xD` (十进制 13)。  因此，这条指令是将初始值 `0xD` 加到数组元素的总和上。

**Line 11: `ret`**

* **`ret`**:  返回指令。  函数执行结束，程序控制权返回给调用者。
* 在 x86-64 调用约定中，函数的返回值 (对于 `int` 类型) 通常通过 `eax` (或 `rax` for 64-bit int) 寄存器返回。  由于最终结果存储在 `eax` 中，所以 `eax` 寄存器中的值就是函数 `foo` 的返回值。

**总结:**

这段汇编代码使用 SSE 指令集和 XMM 寄存器，巧妙地利用 SIMD 并行计算来高效地计算数组元素的总和。  它没有使用传统的循环迭代，而是：

1. **将整个数组加载到 XMM 寄存器 `xmm0` 中，实现数据打包。**
2. **通过 `psrldq` (移位) 和 `paddd` (打包加法) 指令，在 XMM 寄存器内部进行多次并行加法操作，逐步累加数组元素。**  这种方式被称为 **水平加法 (horizontal addition)** 或 **成对求和 (pairwise summation)**。
3. **最终，数组元素的总和被累加到 XMM 寄存器的最低位。**
4. **使用 `movd` 指令将总和提取到通用寄存器 `eax` 中。**
5. **加上初始值 13 (0xD)。**
6. **通过 `eax` 寄存器返回最终结果。**

**优化效果:**

通过使用 SSE 寄存器和 SIMD 指令，编译器将原本需要循环多次的累加操作，优化成了一系列高效的打包操作，显著提高了求和的效率，尤其是在数组元素数量较少且固定 (例如本例中的 4 个元素) 的情况下，这种优化效果更为明显。  这种优化方式充分利用了现代 CPU 的 SIMD 能力，提升了程序性能。

希望这个详细的解析能够帮助您理解这段汇编代码以及 XMM 寄存器在其中的作用！如果您还有任何问题，请随时提出。

> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
4

I am trying to understand how the volatile keyword works in C++.

I had a look at What kinds of optimizations does 'volatile' prevent in C++?. Looking at the accepted answer, it looks like volatile disables two kinds of optimizations

Prevents compilers from caching the value in a register.
Optimizing away accesses to that value when they seem unnecessary from the point of view of your program.
I found similar information at The as-if rule:

Accesses (reads and writes) to volatile objects occur strictly according to the semantics of the expressions in which they occur. In particular, they are not reordered with respect to other volatile accesses on the same thread.

I wrote a simple C++ program that sums all the values in an array to compare the behaviour of plain ints vs. volatile ints. Note that the partial sums are not volatile.

The array consists of unqualified ints.

int foo(const std::array<int, 4>& input)
{
    auto sum = 0xD;
    for (auto element : input)
    {
        sum += element;
    }
    return sum;
}
The array consists of volatile ints:

int bar(const std::array<volatile int, 4>& input)
{
    auto sum = 0xD;
    for (auto element : input)
    {
        sum += element;
    }
    return sum;
}
When I look at the generated assembly code, SSE registers are used only in the case of plain ints. From what little I understand, the code using SSE registers is neither optimizing away the reads nor reordering them across each other. The loop is unrolled, so there aren't any branches either. The only reason I can explain why the code generation is different is: can the volatile reads be reordered before the accumulation happens? Clearly, sum is not volatile. If such reordering is bad, is there a situation/example that can illustrate the issue?

Code generated using Clang 9:

foo(std::array<int, 4ul> const&):                # @foo(std::array<int, 4ul> const&)
        movdqu  (%rdi), %xmm0
        pshufd  $78, %xmm0, %xmm1       # xmm1 = xmm0[2,3,0,1]
        paddd   %xmm0, %xmm1
        pshufd  $229, %xmm1, %xmm0      # xmm0 = xmm1[1,1,2,3]
        paddd   %xmm1, %xmm0
        movd    %xmm0, %eax
        addl    $13, %eax
        retq
bar(std::array<int volatile, 4ul> const&):               # @bar(std::array<int volatile, 4ul> const&)
        movl    (%rdi), %eax
        addl    4(%rdi), %eax
        addl    8(%rdi), %eax
        movl    12(%rdi), %ecx
        leal    (%rcx,%rax), %eax
        addl    $13, %eax
        retq
好的 那回到我们关注的问题上 也就是volatile关键词的使用上
这是stackoverflow的一个用户发布的一个问题
他利用clang编译器和volatile关键词观察其对一个简单的求和代码的影响
首先是 我目前搞不懂他的问题是啥 你能先对他所提问的问题给我详细解释一下是怎么回事吗?

## ChatGPT:
好的，当然。 让我们一起深入分析 Stack Overflow 上的这个问题，理解提问者的困惑以及 `volatile` 关键字在代码优化中的作用。

**提问者的问题核心：**

提问者想通过一个简单的数组求和的例子，来观察 `volatile` 关键字如何影响编译器的代码优化，特别是关于 "阻止寄存器缓存" 和 "阻止优化掉不必要的访问" 这两个方面。

他编写了两个几乎相同的 C++ 函数 `foo` 和 `bar`，唯一的区别在于：

* **`foo` 函数处理的是普通的 `int` 数组。**
* **`bar` 函数处理的是 `volatile int` 数组。**

然后，他使用 Clang 9 编译器，并查看了这两个函数在默认优化级别下的汇编代码输出。  他观察到的关键现象是：

* **对于 `foo` 函数 (普通 `int`)，编译器生成了使用了 SSE 寄存器和 SSE 指令的代码。**  这表明编译器进行了向量化优化，利用 SIMD (单指令多数据) 技术并行处理多个整数的加法。
* **对于 `bar` 函数 (`volatile int`)，编译器生成的代码是标量代码，没有使用 SSE 寄存器，而是使用了普通的通用寄存器 (如 `eax`, `ecx`) 和标量加法指令 (`addl`)。**  这意味着编译器没有对 `bar` 函数进行向量化优化。

**提问者的困惑点可以归纳为：**

1. **为什么 `volatile` 阻止了 SSE 优化？**  提问者理解 `volatile` 会阻止寄存器缓存和优化掉不必要的访问，但他不明白为什么这会导致编译器完全放弃使用 SSE 寄存器进行向量化求和。  SSE 代码看起来并没有违反 `volatile` 的这两个原则，因为 SSE 代码仍然会访问内存 (通过 `movdqu` 加载数据)，并且并没有优化掉任何读取操作，只是将读取和加法操作并行化了。

2. **关于 `volatile` 访问的重排序问题。** 提问者猜测，也许是因为 `volatile` 变量的读取操作不能被重排序到 "累加" 操作之前，而 SSE 的向量化优化可能涉及到某种程度的指令重排，所以编译器为了保证 `volatile` 的语义，就放弃了 SSE 优化。  但他不确定这种重排序是否真的会带来问题，以及在什么情况下会出问题。  他希望找到一个具体的例子来解释 `volatile` 访问重排序可能导致的问题。

**简单来说，提问者想知道：**

* **为什么编译器对 `volatile int` 数组求和不使用 SSE 优化？**
* **`volatile` 阻止优化的深层原因是什么？仅仅是寄存器缓存和优化掉访问吗？还是有更细致的指令重排序方面的考虑？**
* **`volatile` 访问重排序会造成什么实际问题？能否给出一个具体的例子？**

为了解答提问者的问题，我们需要从以下几个方面进行解释：

1. **SSE 优化的本质和 `volatile` 的冲突。**
2. **`volatile` 关键字的核心语义：顺序性和可见的副作用。**
3. **`volatile` 阻止优化的真正原因：不仅仅是寄存器缓存和访问优化，更重要的是保证严格的内存访问顺序和每次访问的可见性。**
4. **提供一个 `volatile` 访问重排序可能导致问题的具体例子，强调 `volatile` 在特定场景下的必要性。**

接下来，我会逐步展开这些解释，帮助您理解提问者的问题以及 `volatile` 关键字的精髓。

> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
The volatile keyword in C++ was inherited it from C, where it was intended as a general catch-all to indicate places where a compiler should allow for the possibility that reading or writing an object might have side-effects it doesn't know about. Because the kinds of side-effects that could be induced would vary among different platforms, the Standard leaves the question of what allowances to make up to compiler writers' judgments as to how they should best serve their customers.

Microsoft's compilers for the 8088/8086 and later x86 have for decades been designed to support the practice of using volatile objects to build a mutex which guards "ordinary" objects. As a simple example: if thread 1 does something like:

ordinaryObject = 23;
volatileFlag = 1;
while(volatileFlag)
  doOtherStuffWhileWaiting();
useValue(ordinaryObject);
and thread 2 periodically does something like:

if (volatileFlag)
{
  ordinaryObject++;
  volatileFlag=0;
}
then the accesses to volatileFlag would serve as a warning to Microsoft's compilers that they should refrain from making assumptions about how any preceding actions on any objects would interact with later actions. This pattern has been followed with the volatile qualifiers in other languages like C#.

Unfortunately, neither clang nor gcc includes any option to treat volatile in such a fashion, opting instead to require that programmers use compiler-specific intrinsics to yield the same semantics that Microsoft could achieve using only the Standard keyword volatile that was intended to be suitable for such purposes [according to the authors of the Standard, "A volatile object is also an appropriate model for a variable shared among multiple processes."--see http://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf p. 76 ll. 25-26]

Share
Edit
Follow
edited Oct 5, 2019 at 23:10
answered Oct 5, 2019 at 21:45
supercat's user avatar
supercat
81.2k99 gold badges176176 silver badges222
这是同一个stackoverflow帖子下的一个answer
refrain from making assumptions about how any preceding actions on any objects would interact with later actions Hmm, I thought C++ 11 memory model solved this issue. – 
Empty Space
 CommentedOct 5, 2019 at 21:58 
4
Volatile is essential for things like reading from the same address mapped to IO multiple times in a row. IO devices may be set up to clock in sequential data chunks on each read and w/o volatile a program can assume that reading from the same address not otherwise modified by the program will not change value. It should not be used for multithreading which has related, but not identical semantics. – 
doug
 CommentedOct 5, 2019 at 22:19
@doug: Can C++11 atomics be fully supported in conforming fashion by freestanding implementations that don't know about any OS that might be used on the target? – 
supercat
 CommentedOct 5, 2019 at 22:27
@doug: Are you saying that the authors of the Standard were wrong when they wrote the above quote? – 
supercat
 CommentedOct 5, 2019 at 22:28
You might want to attribute the source of "[according to the authors of the Standard, "A volatile object is also an appropriate model for a variable shared among multiple processes."]". There is no such statement in any of the C++ standards. Bear in mind that a significant amount of older Microsoft documentation (less so with current documentation) described Microsoft-specific features as being standard C++. – 
Peter
 CommentedOct 5, 2019 at 23:05
@Peter: Citation added. – 
supercat
 CommentedOct 5, 2019 at 23:10
@supercat. Atomics with mutexes, and volatile have different semantics. Microsoft's compilers have an option that enables/disables functionality that makes them useful for multithreading or narrows the semantics. Volatile remains useful for I/O and voids the assumption, useful in optimization, that multiple reads from the same address will return the same value for either option in the Microsoft compiler. – 
doug
 CommentedOct 5, 2019 at 23:11
1
@doug: Microsoft's compilers predated the C and C++ Standards, and nothing in the published Rationale for the Standard indicates any intention that implementations for similar platforms shouldn't be expected to continue behaving as existing compilers for such platforms did in the days before the Standard. – 
supercat
 CommentedOct 5, 2019 at 23:14
2
@supercat, Yes, that isn't the C++11 or greater standard. it's an early C standard and one I wrote a lot of code for at the time. Modern C++ had to address issues due to cache layers and multiple CPUs that earlier C and C++ standards using only volatile were inadequate for without impairing efficiency. A good thing IMO. OTOH, I've never had code that had race condition problems using C++11 or higher and multicore hardware. Volatile alone just doesn't cut it in these environments. At least without impairing performance which the default Microsoft does. – 
doug
 CommentedOct 5, 2019 at 23:25 
@doug: What is the advantage of weakening the meaning of an existing keyword and breaking existing code, versus adding a syntax that would waive some semantic guarantees? – 
supercat
 CommentedOct 5, 2019 at 23:35
@doug: Also, would the C++ Standard allow freestanding implementations for all platforms to support atomics in an OS-agnostic fashion? From what I can tell, at least the C Standard would require that OS-agnostic freestanding implementations accept code that attempts atomic operations on all types, even though on many environments it would be impossible for an OS-agnostic implementation to do so usefully. – 
supercat
 CommentedOct 5, 2019 at 23:39
@supercat Existing code? C++ had no concept of multithreading. It did have a concept of how optimization could remove seemingly redundant reads/writes in a single thread of execution. That's what volatile addressed. Multithreading was done outside the C++ specification until C++11 by compiler extensions and OS specific functions. – 
doug
 CommentedOct 5, 2019 at 23:40
Accepting for the first part of the answer. I am still not convinced about the last part because of the ongoing discussion and also that C standard was referenced and not C++. Although C++ inherits volatile from C, its meaning may have changed post C++ 11. – 
Empty Space
 CommentedOct 6, 2019 at 6:36 
@doug: People have been writing C code which involved interactions between asynchronous execution contexts since before the first C Standard was published, and even before the volatile keyword was invented. The C language does not provide any mechanism by which such contexts may be created, but many execution environments do. Prior to the introduction of volatile, implementations intended to be suitable for use with such code couldn't do much optimization, but the volatile keyword was added as a simple means of making such code compatible with reasonable optimizations. – 
supercat
 CommentedOct 6, 2019 at 18:29
1
@doug: In many cases, compilers didn't need volatile as an indication that they should behave cautiously around code that used I/O registers, since such code would often access pointers that were freshly cast from integers. Since programs would seldom form pointers from integers except in situations that involved doing something "weird", a compiler that made no effort to infer anything about the provenance of such pointers would naturally handle I/O registers just fine even without volatile. – 
supercat
 CommentedOct 6, 2019 at 18:33
1
@TrickorTreat: The authors of the C89 Standard didn't think it necessary to specify that implementations intended for various purposes should provide volatile semantics suitable for those purpose because it seemed obvious. Adding more detail to later standards would have required a consensus, which was blocked by compiler writers who failed to accept things the C89 Committee had thought obvious enough to go without saying. – 
supercat
 CommentedOct 6, 2019 at 18:44
@supercat Right. Good history of the way C/C++ had workarounds based on use cases and not pure language specs. The newer language now includes a set of tools for creating efficient, multi-threaded code, and handling mem mapped IO w/o assumptions about what the compiler will do. More portable. OTOH, I've written a lot of code with pre-standard C/C++. The first step was always to understand the compiler and do any workarounds if needed. Not a big problem but one that is no longer needed in most cases. – 
doug
 CommentedOct 6, 2019 at 19:20
1
@doug: What I find annoying (and IMHO indefensible) is the way the maintainers of clang and gcc assume the purpose of the Standard was to deprecate pre-standard ways of accomplishing things for which support wasn't mandated, despite the fact that the authors of the Standard have expressly stated that they did not intend to demean programs that were useful but happened not to be portable. A compiler with a simple optimizer that was extremely cautious around places where there was evidence of "strangeness" would be much more useful than one with a fancier optimizer that has to be disabled. – 
supercat
 CommentedOct 6, 2019 at 21:13
1
@supercat True. There's a lot of old code that takes advantage of compiler smarts. Even though the language didn't specifically support multithreading C++ was still heavily used. People learned what their compilers did and found ways to accomplish goals. I guess it's up to the compiler maintainers to evolve things based on their user's needs. Microsoft's user base has a ton of these older code bases. Probably why their volatile has expanded semantics unless one opts for the more limited. GCC Clang, etc. could support this too, and make things easier for porting old bases. Oh well. – 
doug
 CommentedOct 6, 2019 at 21:40
@doug: The issue isn't generally "taking advantage of compiler smarts", but rather the opposite: taking advantage of compilers' lack of inappropriate "cleverness". A compiler that behaves as though it knows nothing about whether actions one one lvalue might affect another may not generate fast code, but it will reliably produce correct code. BTW, feel free to join discussion at chat.stackoverflow.com/rooms/200406/… if you like, – 
supercat
 CommentedOct 6, 2019 at 22:04
这是该answer下的全部评论的讨论(answer的作者和另一个人说了很多东西)
呃  你能帮我仔细捋一下answer以及评论都在说什么嘛?

## ChatGPT:
当然，很乐意为您详细梳理 Stack Overflow 答案及其评论的讨论内容，帮助您理解其中关于 `volatile` 关键字的各种观点。

**答案的核心观点：`volatile` 的历史目的和不同编译器的实现差异**

这个答案的核心论点是，C++ 从 C 语言继承的 `volatile` 关键字，最初的目的是作为一个通用的 "提示" 给编译器，表明对某个对象的读写操作可能具有编译器无法预知的 **副作用 (side-effects)**。  这种副作用的类型会因平台而异，因此 C 标准将如何处理 `volatile` 的决定权留给了编译器编写者，让他们根据客户的需求做出最佳判断。

**Microsoft 编译器的 `volatile` 实现：一种早期的简易同步机制**

答案特别指出，Microsoft 的编译器（针对 8088/8086 和后来的 x86 架构）长期以来一直支持使用 `volatile` 对象来构建一种简易的互斥锁 (mutex)，以保护 "普通" 对象。  答案给出了一个简单的例子：

**线程 1:**

```c++
ordinaryObject = 23;
volatileFlag = 1;
while(volatileFlag)
  doOtherStuffWhileWaiting();
useValue(ordinaryObject);
```

**线程 2 (周期性执行):**

```c++
if (volatileFlag)
{
  ordinaryObject++;
  volatileFlag=0;
}
```

在这个例子中，`volatileFlag` 被用作一个标志。线程 1 设置 `ordinaryObject` 的值，然后设置 `volatileFlag` 为 1，表示数据已准备好。线程 2 周期性地检查 `volatileFlag`，如果为 1，则修改 `ordinaryObject` 并将 `volatileFlag` 重置为 0。

**Microsoft 编译器如何理解 `volatile` 在这种场景下的作用：**

Microsoft 编译器将 `volatile` 访问视为一个警告，告诉编译器在处理 `volatileFlag` 的访问时，**不要对任何先前的操作（例如对 `ordinaryObject` 的赋值）与后续操作之间的交互做出任何假设**。  这意味着，编译器需要确保：

* 对 `ordinaryObject` 的写入操作在 `volatileFlag = 1;` 之前完成，并且对其他线程可见。
* 线程 1 的 `while(volatileFlag)` 循环会 **真正地** 检查 `volatileFlag` 的内存值，而不是被优化成始终为真的循环。
* 线程 1 在 `useValue(ordinaryObject)` 之前，能够看到线程 2 对 `ordinaryObject` 的修改。

**关键点：Microsoft 的 `volatile` 实现，在一定程度上，被用来弱化编译器的优化，以支持这种简单的、基于标志的线程同步模式。**  这种模式在 C++11 之前的多线程编程中，在缺乏标准线程库的情况下，可能被一些开发者使用。

**GCC 和 Clang 编译器的 `volatile` 实现：更严格的 I/O 和硬件交互**

答案指出，与 Microsoft 不同，GCC 和 Clang 编译器 **没有提供任何选项** 来以 Microsoft 这种方式处理 `volatile`。  相反，GCC 和 Clang 选择了更严格的 `volatile` 语义，主要将其用于：

* **内存映射 I/O (Memory-Mapped I/O):**  访问硬件寄存器或设备内存。
* **与外部硬件或异步事件交互的场景。**

对于 GCC 和 Clang，如果你想实现类似 Microsoft `volatile` 的弱化优化效果，你需要使用 **编译器特定的内建函数 (intrinsics)**，例如内存屏障 (memory barriers) 或原子操作 (atomic operations)。

**标准制定者的意图 (根据答案作者的解读):**

答案作者引用了 C99 标准的Rationale文档，其中提到 "A volatile object is also an appropriate model for a variable shared among multiple processes." (volatile 对象也是在多个进程之间共享的变量的合适模型)。  答案作者认为，这表明标准制定者最初的意图是希望 `volatile` 能够足够灵活，以适应各种需要编译器谨慎处理的场景，包括进程间/线程间共享变量的简单同步。

**评论区的讨论要点：**

评论区围绕着答案展开了更深入的讨论，主要涉及以下几个方面：

1. **C++11 内存模型和原子操作 (atomics) 的出现：**  评论指出，C++11 引入了标准的内存模型和原子操作库 (`<atomic>`)，这为多线程编程提供了更强大、更清晰、更标准化的工具。  现代 C++ 中，**原子操作才是线程间同步的正确和推荐方式，`volatile` 不适合用于可靠的线程间通信。**

2. **`volatile` 的主要用途：内存映射 I/O 和硬件交互：**  评论强调，`volatile` 在现代 C++ 中最主要的、也是最合适的用途是处理内存映射 I/O 和与硬件设备交互。  在这些场景中，`volatile` 确保每次读写操作都直接访问硬件寄存器，避免编译器的过度优化导致程序行为不符合预期。

3. **Microsoft `volatile` 的历史遗留问题和兼容性：**  评论提到，Microsoft 编译器的 `volatile` 实现方式，虽然在某些早期场景下可能提供了一定的便利性，但它并不是 C++ 标准所定义的 `volatile` 的本意。  这种实现方式更多是出于对历史代码的兼容性考虑。  现代 Microsoft 编译器也提供了选项，允许开发者选择更符合标准 `volatile` 语义的行为。

4. **标准与实现之间的权衡：**  讨论中也涉及到编译器设计者在实现标准时需要做的权衡。  GCC 和 Clang 选择了更严格、更符合现代多线程编程理念的 `volatile` 语义，而 Microsoft 为了兼容旧代码，可能在 `volatile` 的语义上做了一些扩展。

5. **"过于聪明" 的编译器 vs. "谨慎" 的编译器：**  答案作者在评论中表达了一种观点，认为在某些情况下，一个 "不太聪明"、优化策略更保守的编译器，可能比一个 "过于聪明"、激进优化的编译器更可靠。  特别是在处理底层硬件交互或一些 "非典型" 的代码模式时，过于激进的优化可能会导致意想不到的问题。  `volatile` 在某种程度上就是为了限制编译器的 "聪明才智"，让编译器在某些特定场景下保持 "谨慎"。

**总结评论区讨论的核心观点：**

* **现代 C++ 中，`volatile` 的主要用途是内存映射 I/O 和硬件交互，而不是线程间同步。**
* **C++11 原子操作是线程间同步的正确和标准方法，`volatile` 不适合用于此目的。**
* **Microsoft 编译器对 `volatile` 的早期实现方式是一种历史遗留问题，与标准 `volatile` 的本意有所偏差。**
* **GCC 和 Clang 选择了更严格、更符合标准的 `volatile` 语义。**

**回到最初的问题：为什么 `volatile` 阻止了 SSE 优化？**

结合以上分析，我们可以更好地理解为什么编译器对 `volatile int` 数组求和不使用 SSE 优化：

* **`volatile` 的核心语义是保证内存访问的顺序性和可见的副作用。**  SSE 向量化优化，虽然在单线程环境下通常不会改变程序的最终结果，但它可能会改变内存访问的 **具体顺序** 和 **时机**。  例如，SSE 优化可能会将多次内存读取操作合并成一次 128 位读取，或者将读取操作提前到循环之前。  对于 `volatile` 变量，编译器必须严格按照代码的顺序执行内存访问，并确保每次访问都具有外部可见的副作用。  SSE 优化的指令重排和批量操作可能与 `volatile` 的这些语义要求相冲突。

* **编译器为了保守起见，在遇到 `volatile` 变量时，会选择放弃激进的优化策略，例如 SSE 向量化。**  即使在某些情况下，SSE 优化可能看起来 "无害"，但编译器为了保证 `volatile` 的严格语义，最安全的选择就是禁用可能导致指令重排或内存访问模式改变的优化。

* **GCC/Clang 倾向于遵循更严格的 `volatile` 语义，将其主要用于硬件交互等底层场景。**  在这些场景中，内存访问的顺序和时机可能至关重要，任何形式的优化都可能导致程序行为异常。  因此，对于 `volatile` 变量，GCC/Clang 的编译器会非常保守，避免任何可能改变内存访问行为的优化，包括 SSE 向量化。

希望这个详细的梳理和分析能够帮助您理解 Stack Overflow 答案和评论的讨论内容，以及 `volatile` 关键字在 C++ 中的复杂性和不同编译器的实现差异。

> provided by [EasyChat](https://site.eqing.tech/)

# 
C++ volatile和原子操作

## 用户:
refrain from making assumptions about how any preceding actions on any objects would interact with later actions Hmm, I thought C++ 11 memory model solved this issue. – 
Empty Space
 CommentedOct 5, 2019 at 21:58 
4
Volatile is essential for things like reading from the same address mapped to IO multiple times in a row. IO devices may be set up to clock in sequential data chunks on each read and w/o volatile a program can assume that reading from the same address not otherwise modified by the program will not change value. It should not be used for multithreading which has related, but not identical semantics. – 
doug
 CommentedOct 5, 2019 at 22:19
@doug: Can C++11 atomics be fully supported in conforming fashion by freestanding implementations that don't know about any OS that might be used on the target? – 
supercat
 CommentedOct 5, 2019 at 22:27
@doug: Are you saying that the authors of the Standard were wrong when they wrote the above quote? – 
supercat
 CommentedOct 5, 2019 at 22:28
You might want to attribute the source of "[according to the authors of the Standard, "A volatile object is also an appropriate model for a variable shared among multiple processes."]". There is no such statement in any of the C++ standards. Bear in mind that a significant amount of older Microsoft documentation (less so with current documentation) described Microsoft-specific features as being standard C++. – 
Peter
 CommentedOct 5, 2019 at 23:05
@Peter: Citation added. – 
supercat
 CommentedOct 5, 2019 at 23:10
@supercat. Atomics with mutexes, and volatile have different semantics. Microsoft's compilers have an option that enables/disables functionality that makes them useful for multithreading or narrows the semantics. Volatile remains useful for I/O and voids the assumption, useful in optimization, that multiple reads from the same address will return the same value for either option in the Microsoft compiler. – 
doug
 CommentedOct 5, 2019 at 23:11
1
@doug: Microsoft's compilers predated the C and C++ Standards, and nothing in the published Rationale for the Standard indicates any intention that implementations for similar platforms shouldn't be expected to continue behaving as existing compilers for such platforms did in the days before the Standard. – 
supercat
 CommentedOct 5, 2019 at 23:14
2
@supercat, Yes, that isn't the C++11 or greater standard. it's an early C standard and one I wrote a lot of code for at the time. Modern C++ had to address issues due to cache layers and multiple CPUs that earlier C and C++ standards using only volatile were inadequate for without impairing efficiency. A good thing IMO. OTOH, I've never had code that had race condition problems using C++11 or higher and multicore hardware. Volatile alone just doesn't cut it in these environments. At least without impairing performance which the default Microsoft does. – 
doug
 CommentedOct 5, 2019 at 23:25 
@doug: What is the advantage of weakening the meaning of an existing keyword and breaking existing code, versus adding a syntax that would waive some semantic guarantees? – 
supercat
 CommentedOct 5, 2019 at 23:35
@doug: Also, would the C++ Standard allow freestanding implementations for all platforms to support atomics in an OS-agnostic fashion? From what I can tell, at least the C Standard would require that OS-agnostic freestanding implementations accept code that attempts atomic operations on all types, even though on many environments it would be impossible for an OS-agnostic implementation to do so usefully. – 
supercat
 CommentedOct 5, 2019 at 23:39
@supercat Existing code? C++ had no concept of multithreading. It did have a concept of how optimization could remove seemingly redundant reads/writes in a single thread of execution. That's what volatile addressed. Multithreading was done outside the C++ specification until C++11 by compiler extensions and OS specific functions. – 
doug
 CommentedOct 5, 2019 at 23:40
Accepting for the first part of the answer. I am still not convinced about the last part because of the ongoing discussion and also that C standard was referenced and not C++. Although C++ inherits volatile from C, its meaning may have changed post C++ 11. – 
Empty Space
 CommentedOct 6, 2019 at 6:36 
@doug: People have been writing C code which involved interactions between asynchronous execution contexts since before the first C Standard was published, and even before the volatile keyword was invented. The C language does not provide any mechanism by which such contexts may be created, but many execution environments do. Prior to the introduction of volatile, implementations intended to be suitable for use with such code couldn't do much optimization, but the volatile keyword was added as a simple means of making such code compatible with reasonable optimizations. – 
supercat
 CommentedOct 6, 2019 at 18:29
1
@doug: In many cases, compilers didn't need volatile as an indication that they should behave cautiously around code that used I/O registers, since such code would often access pointers that were freshly cast from integers. Since programs would seldom form pointers from integers except in situations that involved doing something "weird", a compiler that made no effort to infer anything about the provenance of such pointers would naturally handle I/O registers just fine even without volatile. – 
supercat
 CommentedOct 6, 2019 at 18:33
1
@TrickorTreat: The authors of the C89 Standard didn't think it necessary to specify that implementations intended for various purposes should provide volatile semantics suitable for those purpose because it seemed obvious. Adding more detail to later standards would have required a consensus, which was blocked by compiler writers who failed to accept things the C89 Committee had thought obvious enough to go without saying. – 
supercat
 CommentedOct 6, 2019 at 18:44
@supercat Right. Good history of the way C/C++ had workarounds based on use cases and not pure language specs. The newer language now includes a set of tools for creating efficient, multi-threaded code, and handling mem mapped IO w/o assumptions about what the compiler will do. More portable. OTOH, I've written a lot of code with pre-standard C/C++. The first step was always to understand the compiler and do any workarounds if needed. Not a big problem but one that is no longer needed in most cases. – 
doug
 CommentedOct 6, 2019 at 19:20
1
@doug: What I find annoying (and IMHO indefensible) is the way the maintainers of clang and gcc assume the purpose of the Standard was to deprecate pre-standard ways of accomplishing things for which support wasn't mandated, despite the fact that the authors of the Standard have expressly stated that they did not intend to demean programs that were useful but happened not to be portable. A compiler with a simple optimizer that was extremely cautious around places where there was evidence of "strangeness" would be much more useful than one with a fancier optimizer that has to be disabled. – 
supercat
 CommentedOct 6, 2019 at 21:13
1
@supercat True. There's a lot of old code that takes advantage of compiler smarts. Even though the language didn't specifically support multithreading C++ was still heavily used. People learned what their compilers did and found ways to accomplish goals. I guess it's up to the compiler maintainers to evolve things based on their user's needs. Microsoft's user base has a ton of these older code bases. Probably why their volatile has expanded semantics unless one opts for the more limited. GCC Clang, etc. could support this too, and make things easier for porting old bases. Oh well. – 
doug
 CommentedOct 6, 2019 at 21:40
能完整的翻译这段对话嘛?

## ChatGPT:
当然，我很乐意帮你翻译这段对话。

**Empty Space** 评论于 2019年10月5日 21:58
> Hmm, I thought C++ 11 memory model solved this issue. –
> 嗯，我以为 C++11 内存模型解决了这个问题。

**doug** 评论于 2019年10月5日 22:19
> Volatile is essential for things like reading from the same address mapped to IO multiple times in a row. IO devices may be set up to clock in sequential data chunks on each read and w/o volatile a program can assume that reading from the same address not otherwise modified by the program will not change value. It should not be used for multithreading which has related, but not identical semantics. –
> `volatile` 对于诸如连续多次从映射到 IO 的同一地址读取这样的操作至关重要。IO 设备可能被设置为在每次读取时按顺序输入数据块，如果没有 `volatile`，程序可能会假设从同一地址读取（程序本身没有修改过）不会改变值。它不应该用于多线程，因为多线程有相关但并不完全相同的语义。

**supercat** 评论于 2019年10月5日 22:27
> @doug: Can C++11 atomics be fully supported in conforming fashion by freestanding implementations that don't know about any OS that might be used on the target? –
> @doug：C++11 原子操作能否在符合规范的方式下，被那些不了解目标平台上可能使用的任何操作系统的独立实现完全支持？

**supercat** 评论于 2019年10月5日 22:28
> @doug: Are you saying that the authors of the Standard were wrong when they wrote the above quote? –
> @doug：你是说标准制定者在写下上面这段引言时是错误的吗？

**Peter** 评论于 2019年10月5日 23:05
> You might want to attribute the source of "[according to the authors of the Standard, "A volatile object is also an appropriate model for a variable shared among multiple processes."]". There is no such statement in any of the C++ standards. Bear in mind that a significant amount of older Microsoft documentation (less so with current documentation) described Microsoft-specific features as being standard C++. –
> 你可能需要注明 "[根据标准制定者，“`volatile` 对象也是在多个进程之间共享的变量的合适模型。”]" 这句话的出处。在任何 C++ 标准中都没有这样的声明。请记住，大量的旧 Microsoft 文档（现在的文档较少）将 Microsoft 特有的功能描述为标准 C++。

**supercat** 评论于 2019年10月5日 23:10
> @Peter: Citation added. –
> @Peter：已添加引用。

**doug** 评论于 2019年10月5日 23:11
> @supercat. Atomics with mutexes, and volatile have different semantics. Microsoft's compilers have an option that enables/disables functionality that makes them useful for multithreading or narrows the semantics. Volatile remains useful for I/O and voids the assumption, useful in optimization, that multiple reads from the same address will return the same value for either option in the Microsoft compiler. –
> @supercat。原子操作（使用互斥锁）和 `volatile` 具有不同的语义。Microsoft 的编译器有一个选项，可以启用/禁用使其对多线程有用的功能，或者缩小其语义范围。`volatile` 对于 I/O 仍然有用，并且否定了在优化中有用的假设，即在 Microsoft 编译器中，无论选择哪个选项，从同一地址多次读取都会返回相同的值。

**supercat** 评论于 2019年10月5日 23:14
> @doug: Microsoft's compilers predated the C and C++ Standards, and nothing in the published Rationale for the Standard indicates any intention that implementations for similar platforms shouldn't be expected to continue behaving as existing compilers for such platforms did in the days before the Standard. –
> @doug：Microsoft 的编译器早于 C 和 C++ 标准，并且在已发布的标准理由中没有任何迹象表明，不应期望类似平台的实现继续像标准之前的那些平台的现有编译器那样运行。

**doug** 评论于 2019年10月5日 23:25
> @supercat, Yes, that isn't the C++11 or greater standard. it's an early C standard and one I wrote a lot of code for at the time. Modern C++ had to address issues due to cache layers and multiple CPUs that earlier C and C++ standards using only volatile were inadequate for without impairing efficiency. A good thing IMO. OTOH, I've never had code that had race condition problems using C++11 or higher and multicore hardware. Volatile alone just doesn't cut it in these environments. At least without impairing performance which the default Microsoft does. –
> @supercat，是的，那不是 C++11 或更高版本的标准。那是一个早期的 C 标准，我当时为它编写了很多代码。现代 C++ 必须解决由于缓存层和多 CPU 导致的问题，而早期的 C 和 C++ 标准仅使用 `volatile` 在不损害效率的情况下是不够的。我认为这是一件好事。另一方面，我从来没有遇到过使用 C++11 或更高版本以及多核硬件的代码出现竞争条件问题。仅靠 `volatile` 在这些环境中是不够的。至少在不损害性能的情况下，而默认的 Microsoft 编译器会这样做。

**supercat** 评论于 2019年10月5日 23:35
> @doug: What is the advantage of weakening the meaning of an existing keyword and breaking existing code, versus adding a syntax that would waive some semantic guarantees? –
> @doug：削弱现有关键字的含义并破坏现有代码，与添加一种可以放弃某些语义保证的语法相比，有什么优势呢？

**supercat** 评论于 2019年10月5日 23:39
> @doug: Also, would the C++ Standard allow freestanding implementations for all platforms to support atomics in an OS-agnostic fashion? From what I can tell, at least the C Standard would require that OS-agnostic freestanding implementations accept code that attempts atomic operations on all types, even though on many environments it would be impossible for an OS-agnostic implementation to do so usefully. –
> @doug：另外，C++ 标准是否允许所有平台的独立实现以与操作系统无关的方式支持原子操作？据我所知，至少 C 标准会要求与操作系统无关的独立实现接受尝试对所有类型进行原子操作的代码，即使在许多环境中，与操作系统无关的实现不可能有效地做到这一点。

**doug** 评论于 2019年10月5日 23:40
> @supercat Existing code? C++ had no concept of multithreading. It did have a concept of how optimization could remove seemingly redundant reads/writes in a single thread of execution. That's what volatile addressed. Multithreading was done outside the C++ specification until C++11 by compiler extensions and OS specific functions. –
> @supercat 现有代码？C++ 没有多线程的概念。它确实有一个关于优化如何删除单线程执行中看似冗余的读取/写入的概念。这就是 `volatile` 要解决的问题。在 C++11 之前，多线程是通过编译器扩展和操作系统特定功能在 C++ 规范之外完成的。

**Empty Space** 评论于 2019年10月6日 6:36
> Accepting for the first part of the answer. I am still not convinced about the last part because of the ongoing discussion and also that C standard was referenced and not C++. Although C++ inherits volatile from C, its meaning may have changed post C++ 11. –
> 接受答案的第一部分。由于正在进行的讨论，以及引用的是 C 标准而不是 C++ 标准，我仍然对最后一部分不太信服。虽然 C++ 从 C 继承了 `volatile`，但其含义可能在 C++11 之后发生了变化。

**supercat** 评论于 2019年10月6日 18:29
> @doug: People have been writing C code which involved interactions between asynchronous execution contexts since before the first C Standard was published, and even before the volatile keyword was invented. The C language does not provide any mechanism by which such contexts may be created, but many execution environments do. Prior to the introduction of volatile, implementations intended to be suitable for use with such code couldn't do much optimization, but the volatile keyword was added as a simple means of making such code compatible with reasonable optimizations. –
> @doug：自从第一个 C 标准发布之前，甚至在 `volatile` 关键字发明之前，人们就已经在编写涉及异步执行上下文之间交互的 C 代码。C 语言没有提供任何创建此类上下文的机制，但许多执行环境提供了。在引入 `volatile` 之前，旨在适用于此类代码的实现无法进行太多优化，但添加 `volatile` 关键字是一种使此类代码与合理的优化兼容的简单方法。

**supercat** 评论于 2019年10月6日 18:33
> @doug: In many cases, compilers didn't need volatile as an indication that they should behave cautiously around code that used I/O registers, since such code would often access pointers that were freshly cast from integers. Since programs would seldom form pointers from integers except in situations that involved doing something "weird", a compiler that made no effort to infer anything about the provenance of such pointers would naturally handle I/O registers just fine even without volatile. –
> @doug：在许多情况下，编译器不需要 `volatile` 来指示它们应该在处理使用 I/O 寄存器的代码时谨慎行事，因为此类代码通常会访问从整数新近强制转换而来的指针。由于程序很少会从整数形成指针，除非在涉及做一些“奇怪”事情的情况下，因此即使没有 `volatile`，一个不努力推断此类指针来源的编译器也会自然而然地很好地处理 I/O 寄存器。

**supercat** 评论于 2019年10月6日 18:44
> @TrickorTreat: The authors of the C89 Standard didn't think it necessary to specify that implementations intended for various purposes should provide volatile semantics suitable for those purpose because it seemed obvious. Adding more detail to later standards would have required a consensus, which was blocked by compiler writers who failed to accept things the C89 Committee had thought obvious enough to go without saying. –
> @TrickorTreat：C89 标准的制定者认为没有必要明确规定，旨在用于各种目的的实现应该提供适合这些目的的 `volatile` 语义，因为这似乎是显而易见的。在后来的标准中添加更多细节将需要达成共识，但这被那些未能接受 C89 委员会认为显而易见以至于无需赘言的事情的编译器编写者所阻止。

**doug** 评论于 2019年10月6日 19:20
> @supercat Right. Good history of the way C/C++ had workarounds based on use cases and not pure language specs. The newer language now includes a set of tools for creating efficient, multi-threaded code, and handling mem mapped IO w/o assumptions about what the compiler will do. More portable. OTOH, I've written a lot of code with pre-standard C/C++. The first step was always to understand the compiler and do any workarounds if needed. Not a big problem but one that is no longer needed in most cases. –
> @supercat 对。很好地回顾了 C/C++ 基于用例而不是纯粹的语言规范进行变通的历史。较新的语言现在包含一套工具，用于创建高效的多线程代码，并在不假设编译器将做什么的情况下处理内存映射 IO。更具可移植性。另一方面，我用标准之前的 C/C++ 编写了很多代码。第一步总是了解编译器，并在需要时进行任何变通。这不是一个大问题，但在大多数情况下不再需要了。

**supercat** 评论于 2019年10月6日 21:13
> @doug: What I find annoying (and IMHO indefensible) is the way the maintainers of clang and gcc assume the purpose of the Standard was to deprecate pre-standard ways of accomplishing things for which support wasn't mandated, despite the fact that the authors of the Standard have expressly stated that they did not intend to demean programs that were useful but happened not to be portable. A compiler with a simple optimizer that was extremely cautious around places where there was evidence of "strangeness" would be much more useful than one with a fancier optimizer that has to be disabled. –
> @doug：我感到恼火（并且恕我直言，站不住脚）的是 clang 和 gcc 的维护者假设标准的目的是要废弃那些没有强制支持的、在标准之前实现某些目标的方式，尽管标准的制定者明确表示他们无意贬低那些有用但不巧不便携的程序。一个具有简单优化器，在有“奇怪”迹象的地方极其谨慎的编译器，将比一个必须禁用的更花哨的优化器更有用。

**doug** 评论于 2019年10月6日 21:40
> @supercat True. There's a lot of old code that takes advantage of compiler smarts. Even though the language didn't specifically support multithreading C++ was still heavily used. People learned what their compilers did and found ways to accomplish goals. I guess it's up to the compiler maintainers to evolve things based on their user's needs. Microsoft's user base has a ton of these older code bases. Probably why their volatile has expanded semantics unless one opts for the more limited. GCC Clang, etc. could support this too, and make things easier for porting old bases. Oh well. –
> @supercat 确实如此。有很多旧代码利用了编译器的智能。即使该语言没有明确支持多线程，C++ 仍然被大量使用。人们了解了他们的编译器所做的事情，并找到了实现目标的方法。我想这取决于编译器维护者根据用户的需求来发展事物。Microsoft 的用户群拥有大量的旧代码库。这可能就是为什么他们的 `volatile` 具有扩展的语义，除非选择更有限的语义。GCC、Clang 等也可以支持这一点，并使移植旧代码库更容易。唉，就这样吧。

希望这个翻译对您有帮助！

> provided by [EasyChat](https://site.eqing.tech/)
