+++
title = "C++ Core Guidelines 命名与布局规则"
date = 2020-12-20 00:30:45
slug = "202012200030"

[taxonomies]
tags = ["C++"]
+++

C++ Core Guidelines 是现代 C++（目前是 C++17）的一套核心指导方针，考虑了未来的增强与 ISO 技术规格。

<!-- more -->

本文翻译了 NL（Naming and layout rules）部分，原文在这里 [C++ Core Guidelines Naming and layout rules](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#nl-naming-and-layout-rules)

统一的命名与布局非常有帮助。因为它最小化了「我的风格比你的风格更好」的争论。然而，目前有非常非常多的不同风格，人们对于它们也很热情（积极或消极）。并且，大多数现实世界的项目会包括许多不同来源的代码，因此对所有的代码使用单一风格进行标准化通常是不可能的。在用户对于指导方针的多次要求之后，ISO C++ 提出了一套规则供用户在没有更好的想法的情况下使用，但是真正的目标是统一风格而不是要提出某个特定的规则。

## NL.1 如果代码能表达清楚就不要用注释

**理由** IDE 不会读注释。注释不够代码精确。注释不会更新得像代码一样连贯。

**反面例子**

```cpp
auto x = m * v1 + vv;   // multiply m with v1 and add the result to vv
```

**强制措施** 去写一个理解英语口语的 AI 程序，看看能不能比 C++ 更容易解读

## NL.2 在注释中表达意图

**理由** 代码描述的是具体做了什么事情，而不会描述这么做的目的。通常来说，比起代码实现，使用注释能够更加清晰和准确地表达意图。

**例子**

```cpp
void stable_sort(Sortable& c)
    // sort c in the order determined by <, keep equal elements (as defined by ==) in
    // their original relative order
{
    // ... quite a few lines of non-trivial code ...
}
```

**注意** 如果代码的行为跟注释不一致，错误可能在于注释也可能在于代码

## NL.3 保持注释的简明扼要

**理由** 冗长的注释不易理解，并且因为注释是穿插在源代码中的，反而导致代码不易阅读。

**注意** 使用通俗易懂的英语。我可能丹麦语更熟练，但大多数程序员也许不会；我的代码的维护者可能也不会。避免一些不正规的缩写（原文是指 SMS 用语），并且审查你的语法、标点和大小写。注释的目标是专业化而不是为了看起来酷炫。

**强制措施** 不可能。

## NL.4 维护统一的缩进风格

**理由** 可读性。避免一些愚蠢的错误。

**反面例子**

```cpp
int i;
for (i = 0; i < max; ++i); // bug waiting to happen
if (i == j)
    return i;
```

**注意** 缩进 `if (...)`、`for (...)`、`while (...)` 之后的语句是一个好的想法。

```cpp
if (i < 0) error("negative argument");

if (i < 0)
    error("negative argument");
```

**强制措施** 使用工具

## NL.5 避免把类型编码到命名中

**根本原因** 如果命名中反映的是类型而不是功能，用于提供那种功能的类型就会难以改变。并且，如果变量的类型改变了，使用它的其他代码也需要修改。应当最小化无意的转换

**反面例子**

```cpp
void print_int(int i);
void print_string(const char*);

print_int(1);          // repetitive, manual type matching
print_string("xyzzy"); // repetitive, manual type matching
```

**例子**

```cpp
void print(int i);
void print(string_view);    // also works on any string-like sequence

print(1);              // clear, automatic type matching
print("xyzzy");        // clear, automatic type matching
```

**注意** 把类型编码到命名中不仅冗余而且还难以理解（译者注：这是什么奇怪的行为艺术吗）

```cpp
printS  // print a std::string
prints  // print a C-style string
printi  // print an int
```

需要类似匈牙利命名法来编码类型的技术已经被用于无类型的语言中，但是它在诸如 C++ 这样的强类型且是静态类型的语言中通常是不必要的而且具有极大破坏性，因为这种技术已经过时并且它们还干涉了对语言的有效利用（比如函数重载）

**注意** 一些命名风格使用非常通用（而不是只局限于类型）的前缀来表示变量的通常用途

```cpp
auto p = new User();
auto p = make_unique<User>();
// note: "p" is not being used to say "raw pointer to type User,"
//       just generally to say "this is an indirection"

auto cntHits = calc_total_of_hits(/*...*/);
// note: "cnt" is not being used to encode a type,
//       just generally to say "this is a count of something"
```

这样写不具有破坏性并且没有违反本指导方针因为它没有编码类型信息。

**注意** 一些命名风格区分成员变量和局部变量，可能也有全局变量

```cpp
struct S {
    int m_;
    S(int m) : m_{abs(m)} { }
};
```

这样写不具有破坏性并且没有违反本指导方针因为它没有编码类型信息。

**注意** 像 C++ 一样，一些风格区分类型与非类型。比如类型名的首字母大写而变量和函数不会

```cpp
typename<typename T>
class HashTable {   // maps string to T
    // ...
};

HashTable<int> index;
```

这样写不具有破坏性并且没有违反本指导方针因为它没有编码类型信息。

## NL.7 命名的长度要与它所在的作用域范围成大致的比例

**根本原因** 作用域范围越大，歧义和无意的命名冲突的可能性就越大。

**例子**

```cpp
double sqrt(double x);   // return the square root of x; x must be non-negative

int length(const char* p);  // return the number of characters in a zero-terminated C-style string

int length_of_string(const char zero_terminated_array_of_char[])    // bad: verbose

int g;      // bad: global variable with a cryptic name

int open;   // bad: global variable with a short, popular name
```

使用 `p` 作为指针，使用 `x` 作为浮点变量是符合习俗的且在一个局部的作用域中不会产生歧义

**强制措施** 谁知道呢

## NL.8 使用统一的命名风格

**根本原因** 统一的命名风格提高可读性。

**注意** 命名风格有很多种且当你使用多个类库的时候，你不可能遵从它们全部的命名习惯。选择一个主体风格，导入的其他类库就保持它们原本的命名风格。

**例子** ISO Standard：使用小写字母和数字，下划线分隔单词

- `int`
- `vector`
- `my_map`

避免双下划线 `__`。

**例子** [Stroustrup](http://www.stroustrup.com/Programming/PPP-style.pdf)：自定义的类型和 concepts 的首字母大写，其他与 ISO Standard 相同。

- `int`
- `vector`
- `My_map`

**例子** 驼峰：单词的首字母大写

- `int`
- `vector`
- `MyMap`
- `myMap`

有些习惯会把第一个字母也大写，有的不会

**注意** 如果你使用首字母缩写，尽量和其他标识符长度保持统一

```cpp
int mtbf {12};
int mean_time_between_failures {12}; // make up your mind
```

**强制措施** 可以实现，除了使用外部库的情况

## NL.9 只有宏才能使用 `ALL_CAPS` 的形式

**理由** 为了避免混淆宏与其他遵守了作用域和类型规则的命名。

**例子**

```cpp
void f()
{
    const int SIZE{1000};  // Bad, use 'size' instead
    int v[SIZE];
}
```

**注意** 不是宏的符号常量也不能用全大写

```cpp
enum bad { BAD, WORSE, HORRIBLE }; // BAD
```

**强制措施**

- 标记出现了小写字母的宏
- 标记使用 `ALL_CAPS` 的非宏命名

## NL.10 优先考虑 `underscore_style` 命名

**理由** 下划线命名法是最初的 C 和 C++ 风格且它也用于 C++ 标准库

**注意** 这条规则是只有在你没有其他选择的情况下的默认选项。通常来说，你没有选择而且必须遵守一种确定好的风格来保证代码的统一。对于统一的需求可能会与个人喜好冲突。

这只是在你没有限制或者更好的想法的情况的一种推荐。这条规则是在对于指导方针的多次请求后才添加的。

**例子** [Stroustrup](http://www.stroustrup.com/Programming/PPP-style.pdf)：自定义的类型和 concepts 的首字母大写，其他与 ISO Standard 相同。

- `int`
- `vector`
- `My_map`

**强制措施** 不可能。（译者注：谁都没有办法阻止我用驼峰，连 ISO C++ 都不行）

## NL.11 保证 literal 可读性

**理由** 可读性

**例子** 使用数组分隔符来避免大段的数字字符串

```cpp
auto c = 299'792'458; // m/s2
auto q2 = 0b0000'1111'0000'0000;
auto ss_number = 123'456'7890;
```

**例子** 在需要声明的地方使用 literal 后缀

```cpp
auto hello = "Hello!"s; // a std::string
auto world = "world";   // a C-style string
auto interval = 100ms;  // using <chrono>
```

**注意** Literals 不应当作为魔法常量穿插在代码中，但是让它们在被定义的地方保持可读性仍然是一种好想法。在一长串数字的字符串中打错字符是很常见的。

**强制措施** 标记长串的数字。问题在于多长才算长；可能是 7。

## NL.15 尽可能少的使用空格

**理由** 太多的空格使得文本变长而且会分散程序员的注意力。

**反面例子**

```cpp
#include < map >

int main(int argc, char * argv [ ])
{
    // ...
}
```

**例子**

```cpp
#include <map>

int main(int argc, char* argv[])
{
    // ...
}
```

**注意** 一些 IDE 有他们自己的见解并且会添加使人分心的空格。

这只是在你没有限制或者更好的想法的情况的一种推荐。这条规则是在对于指导方针的多次请求后才添加的。

**注意** 我们同样重视合理放置的空格，它能为可读性提供重大的帮助。不要矫枉过正。

## NL.16 使用惯用的类成员声明顺序

**理由** 一个惯用的成员顺序提高可读性。

声明一个类的时候应当使用如下顺序

- 类型：类、enums 和别名（`using`）
- 构造函数、赋值函数、析构函数
- 普通函数
- 数据成员

使用 `public` 到 `protected` 再到 `private` 的顺序。

这只是在你没有限制或者更好的想法的情况的一种推荐。这条规则是在对于指导方针的多次请求后才添加的。

**例子**

```cpp
class X {
public:
    // interface
protected:
    // unchecked function for use by derived class implementations
private:
    // implementation details
};
```

**例子** 有时候成员的默认顺序会跟分离公共接口和实现细节的意愿产生冲突。这种情况下，私有类型和函数可以跟私有数据成员放在一起。（译者注：简单说就是优先考虑访问限定符）

```cpp
class X {
public:
    // interface
protected:
    // unchecked function for use by derived class implementations
private:
    // implementation details (types, functions, and data)
};
```

**反面例子** 避免把同一个访问限定符分散地多次声明在另一种访问限定符之间

```cpp
class X {   // bad
public:
    void f();
public:
    int g();
    // ...
};
```

使用宏来声明成员的分组通常最终会导致违反这些成员声明顺序的规则。但是无论如何，宏还是会隐藏掉最终会表达出来的东西。

**强制措施** 标记推荐的顺序的起点。将会有很多遗留代码不遵守这个规则。

## NL.17 使用源于 K&R 的布局

**理由** 这是最初的 C 和 C++ 布局。它很好地保留了垂直空间。它很好地区分了不同不同的语言结构（比如函数和类）

**注意** 在 C++ 上下文中，这种风格通常被称作 Stroustrup。

这只是在你没有限制或者更好的想法的情况的一种推荐。这条规则是在对于指导方针的多次请求后才添加的。

**例子**

```cpp
struct Cable {
    int x;
    // ...
};

double foo(int x)
{
    if (0 < x) {
        // ...
    }

    switch (x) {
    case 0:
        // ...
        break;
    case amazing:
        // ...
        break;
    default:
        // ...
        break;
    }

    if (0 < x)
        ++x;

    if (x < 0)
        something();
    else
        something_else();

    return some_value;
}
```

注意 `if` 和 `(` 之间的空格

**注意** 对于每个语句使用单独的一行，比如 `if` 的一个分支，还有 `for` 的 body。

**注意** 类和结构体的 `{` 不会另起一行，但是函数的 `{` 会占用单独的一行

**注意** 你自己的用户定义类型应当首字母大写，从而跟标准库区分开来。

**注意** 函数名不要首字母大写

**强制措施** 如果你需要的话，使用 IDE 的自动格式化。

## NL.18 使用 C++ 风格的声明布局

**理由** C 风格的命名布局强调表达式跟语法的使用，然而 C++ 风格强调类型。表达式不会持有引用。

**例子**

```cpp
T& operator[](size_t);   // OK
T &operator[](size_t);   // just strange
T & operator[](size_t);   // undecided
```

这只是在你没有限制或者更好的想法的情况的一种推荐。这条规则是在对于指导方针的多次请求后才添加的。

**强制措施** 由于历史原因，这是不可能的。

## NL.19 避免容易误读的命名

**理由** 可读性。不是每个人都有轻易识别所有字母的屏幕。我们很容易就会误读具有相似拼写或者具有轻微拼写错误的单词。

**例子**

```cpp
int oO01lL = 6; // bad

int splunk = 7;
int splonk = 8; // bad: splunk and splonk are easily confused
```

**强制措施** 谁知道呢

## NL.20 不要把两个语句放同一行

**理由** 可读性。如果两个语句放同一行真的很容易漏看。

**例子**

```cpp
int x = 7; char* p = 29;    // don't
int x = 7; f(x);  ++x;      // don't
```

**强制措施** 很简单

## NL.21 每行只声明一个变量

**理由** 可读性。最小化声明句法的误解。

**注意** 更多细节可以参考 ES.10。

## NL.25 不要把 `void` 放在函数参数里

**理由** 冗余并且只有在考虑 C 的兼容性时才需要这样做。

**例子**

```cpp
void f(void);   // bad

void g();       // better
```

**注意** 甚至连 Dennis Ritchie 都认为 `void f(void)` 是一个非常令人憎恨的东西。当函数原型非常少的时候，你可以为这个 C 的可恨的东西辩护。因此不应当使用：

```cpp
int f();
f(1, 2, "weird but valid C89");   // hope that f() is defined int f(a, b, c) char* c; { /* ... */ }
```

这会导致重大的问题，但是不会在 21 世纪出现，也不会在 C++ 中出现。

## NL.26 使用惯用的 `const` 标记

**理由** 对于更多的程序员而言，他们更熟悉惯用的标记。维持在大范围的代码库中的统一性。

**例子**

```cpp
const int x = 7;    // OK
int const y = 9;    // bad

const int *const p = nullptr;   // OK, constant pointer to constant int
int const *const p = nullptr;   // bad, constant pointer to constant int
```

**注意** 我们当然知道你可能会说把 `const` 放在后面更加的符合逻辑，但是这可能会使得更多的人非常困惑，特别是初学者，他们依赖于使用更加常用的符合习惯的风格的教学材料。

跟之前一样，记住这些命名和布局的规则的目的只是为了保持统一性，不可能符合所有人的审美。

这只是在你没有限制或者更好的想法的情况的一种推荐。这条规则是在对于指导方针的多次请求后才添加的。

**强制措施** 标记出现在类型名之后的 `const`。
