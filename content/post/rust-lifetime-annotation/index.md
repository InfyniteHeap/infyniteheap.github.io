---
title: "Rust生命周期简单理解"
description: "尝试以简单的方式解释Rust生命周期标注概念，帮助Rust程序员理解这一难点。"
slug: "rust-lifetime-annotation"
date: "2025-07-05T19:06:06+08:00"
image: ruan-mei.jpg
categories:
  - 计算机软件
tags:
  - Rust
---

> 本文假设读者已有一定 Rust 编程经验，至少能理解所有权和借用的概念与规则。

Rust 有三大核心概念，分别是所有权、借用和生命周期。而在这三大核心概念中，由于生命周期是程序员接触最少的概念。因此，在这三大核心概念中，了解生命周期的概念，优先性是第一位的......好了，编不下去了。（悲）

谈一下为什么我会写这样一篇文章：虽然学习生命周期的前提是理解所有权和借用，但你仍然需要单独学习生命周期的概念。另外，Rust 官方的教程 The
Book 给出的关于生命周期的解释也存在一些缺陷（这会在下文提及）。因此，我在此分享一下我对生命周期的理解。

（其实，说是分享我的理解，其实是我希望这篇文章能起到抛砖引玉的作用，为读者——也就是你——深入理解生命周期的概念提供一种新思路。）

当然，人非圣贤，孰能无过？限于本人水平，本文一定存在不少谬误。若有发现，请务必批评指出！

## 生命周期的历史

首先，必须指出，“生命周期”的概念本身并非由 Rust 首次提出。事实上，古老若 C 语言，它也有生命周期的概念。但它的生命周期概念并不明显，且仅局限于变量及其所在作用域的关系，即：变量的生命周期会在其离开其所在作用域后结束。

C 语言之后的部分语言加强了生命周期的显式概念，如 C++，其变量的生命周期由其创建和销毁的时机决定。如类的构造函数和析构函数：

```cpp
class ArbitraryClass {
public:
    int a = 0;
    // 这是构造函数
    ArbitraryClass() { int a = 5; };
    // 这是析构函数
    ~ArbitraryClass() {};
};
```

使用`new`和`delete`来分别调用构造函数和析构函数：

```cpp
auto obj = new ArbitraryClass();
delete obj;
```

这样，我们便可以轻松管控变量`obj`的生命周期了：使用`new`函数创建`ArbitraryClass`的对象，随后使用`delete`函数释放它。

但是，
过去的 C++编译器并不会静态检查变量的生命周期，这可能会导致各种常见问题，如内存泄漏或造成迷途指针等，对内存安全造成威胁。

从 C++11 开始，C++引入了智能指针。它能自动决定何时销毁对象，一定程度上解决了上述问题：

```cpp
unique_ptr<ArbitraryClass> obj(new ArbitraryClass());
// 作用域结束时，该指针会被自动释放
```

当然，智能指针也不能避免某些问题，如产生迷途指针：

```cpp
unique_ptr<ArbitraryClass> obj(new ArbitraryClass());
// 由于move转移了指针的所有权，因此，原来的obj就变成了迷途指针
auto obj2 = move(obj);
```

我们可以使用 clangd 或部分 C++编译器（如 gcc）的新版本来分析智能指针释放后的指针调用行为。不幸的是，截至撰写本文时，它们仍然无法识别部分类型的迷途指针调用行为，例如：

```cpp
unique_ptr<ArbitraryClass> obj(new ArbitraryClass());
auto obj2 = move(obj);
// 下面被注释掉的代码能被clangd或gcc正确分析
// auto a = obj;

// 该代码不能被clangd或gcc正确分析
std::cout << obj->a << std::endl;
```

上述代码可以正常编译通过，且在运行时会发生段错误。因此，C++的生命周期机制也并非十分完善。

或许我们还有其它静态检查工具来规避上述错误，但很遗憾，这些工具是完全可选的！这就意味着 C++程序员需要额外花费精力去了解、配置并运行这些工具。而且，上文提到，静态检查工具也不能检测出所有非法操作行为，这还需要程序员再设置一套完备的代码审查流程，而这对于个人开发者而言几乎不可能做到。

Rust 则是在 C++的基础上更上一层楼。它利用所有权和借用来管理每个值（这里不叫“变量”是因为 Rust 的“变量”默认不可变）的生命周期，以此来保证内存安全。

当然，在开始介绍 Rust 生命周期之前，我们需要简单复习一下所有权和借用的概念。

## 所有权&借用

首先简要说一下所有权。

Rust 所有权规则无外乎这三点（严谨起见，此处给出英文原文）：

- Each value in Rust has an owner.
- There can only be one owner at a time.
- When the owner goes out of scope, the value will be dropped.

看起来很简单对不对？但它确实是 Rust 不借助 GC 却能保证内存安全的灵魂！所有代码必须遵循这套规则，否则会遭到编译器无情地打回——甚至 unsafe
Rust 亦是如此！

拥有复制语义的类型（如`i32`）无需担心此问题，因为 Rust 会在需要时自动复制一个值，如：

```rust
let orig = 5;
let orig2 = orig;
println!("{orig}");
```

按照所有权规则，此时的`orig`理应不再持有`5`，编译报错才对，但上述代码却可以正常编译通过！这正是因为`i32`类型实现了`Copy`
trait。程序运行时，Rust 会自动复制一个`5`并赋值给`orig2`，且不会带来任何性能开销。如此，`orig`和`orig2`均持有对 5 的所有权了，代码就能编译通过了。

- 注意：`orig`和`orig2`所持有的`5`的含义并非相同，因为这两个 5 存储在不同的栈帧。由于`i32`类型的值存储在栈上，因此，复制一个`i32`的值，只需要再开辟一个新栈帧即可。这个过程很快，因此这个复制操作不会带来任何性能开销。

接下来再简要说一下借用。

借用，就是向某个值的 owner 借一个值，用完再还给 owner。这和平常人与人之间借东西一样。

当然，开始之前，让我给出一个不等式（不等式做题就是快:D）：

- 拥有所有权 ≠ 一直持有这个值

其实这也符合人与人之间借东西的情景。譬如，你有一把扳手，别人来借，你手上就没有扳手了。倘使这时又有人来借扳手，你只能告诉他你的扳手被借走了，没法再借他一个（你总不能无中生有一把或故意不借给他吧？）。等到借走你扳手的人把它还回来，你便再次持有了它。在此期间，扳手的 owner 一直是你，但你并非每时每刻都持有它。

Rust 中的借用分为不可变借用及可变借用两种。前者一般也被称为“引用”，后者则是用于值的修改：

```rust
let a = 10;
let mut b = 10;
// 此为不可变借用
let ref_a = &a;
// 此为可变借用
let ref_b = &mut b;
let c = 20;
// 可以对不可变值或可变值进行不可变借用，但不可以对不可变值进行可变借用！
// 如以下被注释掉的代码，去掉注释，程序就不能正常编译运行
// let invalid_mutable_borrow = &mut c;

// Rust还支持这种方式的借用声明
let d = 30;
let ref ref_d = d;

*ref_b *= 3;
assert_eq!(b, 30)
```

与上述所有权规则类似，借用也有其规则 ：

- At any given time, you can have either one mutable reference or any number of
  immutable references.
- References must always be valid.

简单复习了一下所有权和借用，接下来就要进入正题了。

## Rust 生命周期

### 导入

事实上，我们一般只会接触到所有权和借用这两大概念，对于生命周期了解甚少，因为 Rust 已经帮我们省略了许多显式生命周期标注工作：

```rust
fn main() {
    let s = String::from("text");
    let s2 = receive_and_return_a_string(&s);
    println!("{s}");
    println!("{s2}")
}

fn receive_and_return_a_string(str: &str) -> &str {
    str
}
```

对于函数`receive_and_return_a_string`，它的完整形式应是这样：

```rust
fn receive_and_return_a_string<'a>(str: &'a str) -> &'a str {
    str
}
```

由于该函数只有一个形参，因此，借用检查器可以轻松推断出，函数返回值的生命周期只与它有关。发生调用行为时，Rust 将字符串`s`的借用传入函数，随后又原封不动地将其传出函数并赋值给`s2`，同时`"text"`没有走到作用域末尾，值仍然有效。于是编译通过，没有任何问题。

然而对于部分带借用的多形参函数而言，借用检查器就有些力不从心了（汗流浃背了吧借用检查器？:D）：

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

代码逻辑简单明了，看似没有任何问题，但编译时就出了问题：

```text
error[E0106]: missing lifetime specifier
  --> src/main.rs:11:33
   |
11 | fn longest(x: &str, y: &str) -> &str {
   |               ----     ----     ^ expected named lifetime parameter
   |
   = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
   |
11 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
   |           ++++     ++          ++          ++

For more information about this error, try `rustc --explain E0106`.
```

包括 The
Book 在内的大部分 Rust 学习书籍都在告诉初学者，添加显式生命周期注解的原因是编译器无法判断要返回`x`和`y`中的哪一个，却并为对此展开详细说明。

这正是本文欲达成的核心目的：从理论角度详细解释为什么需要添加显式生命周期注解！这样做还有一个目的就是，编译器不可能在所有情况都能给出有帮助的错误提示，这就需要我们拥有丰富的生命周期知识来修复此类错误！

接下来，我们一同探讨一下需要显式标注生命周期声明的原因。

### 分析

开始之前，我们需要知道以下几点：

- 显式生命周期注解是一种特殊的泛型。
- 生命周期注解的目的在于帮助借用检查器判断不同值之间的生命周期关系，而不是改变某个值的生命周期。
- 函数在调用时，程序会为其开辟新的作用域空间。这就意味着在一般情况下，函数形参的生命周期与函数本身一致，不受实参影响。

让我们结合具体代码来分析：

```rust
fn main() {
    let x = String::from("abc");
    let a;
    {
        let y = String::from("abcd");
        a = longest(&x, &y);
    }
    println!("{a}")
}

fn longest(first: &str, second: &str) -> &str {
    if first.len() > second.len() {
        first
    } else {
        second
    }
}
```

容易看出，x 和 y 的生命周期是已知的，且`'x`大于`'y`：

```rust
fn main() {
    let x = String::from("abc"); // ---------------|
    let a; //                                      |
    { //                                           |
        let y = String::from("abcd"); // ----|     |
        a = longest(&x, &y); //             'y    'x
    } // ------------------------------------|     |
    println!("{a}") //                             |
} // ----------------------------------------------|
```

`a`的生命周期看起来和`x`一样，但别忘了，它是个借用（根据函数返回值）。由于借用方生命周期必须小于等于出借方生命周期，因此，它的合法生命周期无法提前得知，必须依赖函数对其的赋值才能确定。

再看`longest`函数定义：

```rust
fn longest(first: &str, second: &str) -> &str {
    if first.len() > second.len() {
        first
    } else {
        second
    }
}
```

可以看出，形参只有`first`和`second`这两个对字符串字面量的借用，返回值也是如此。我们知道，调用函数时，程序会在栈内存开辟新的栈帧，将调用的函数压入栈。由于没有显式生命周期注解，因此`first`和`second`的生命周期与函数本身的生命周期一致，这就意味着二者会随着函数本身的析构而析构。

换句话就是，函数实参自带的生命周期信息未能和函数形参的生命周期关联起来。

还记得前面提到的“显式生命周期注解是一种特殊的泛型”这句话吗？在这里就体现出来了。由于向函数传参时也会发生模式匹配，因此，匹配实参时，实参自己的生命周期信息会被函数忽略，只匹配借用本身：

```rust
// 由于没有添加显式生命周期注解，因此，'x和'y被函数忽略
longest(      'x x,         'y y )
                 |             |
longest(first: &str, second: &str)
```

这样，函数外部出借方的销毁与否，函数就无从可知了。因此在返回借用时，a 很有可能拿到一个无效借用，这就违反了 Rust 所有权机制，编译出错。

综上，我们便知晓了`longest`必须添加显式生命周期注解的根本原因：借用检查器无法确定在把借用赋值给`a`时`x`和`y`的生命周期是否仍然有效！详细地说，作为特殊的泛型参数，由于我们没有在函数签名里把这个泛型参数标注出来，因此函数便无法获得`x`和`y`的生命周期信息，进而在向函数传参时，借用检查器便无从得知二者的生命周期关系及其有效期了。

大白话就是，借用检查器是个极度保守派，当`longest`没有生命周期注解时，它会为`longest`形参脑补一个生命周期，但它不知道实参的生命周期的情况，不敢轻举妄动，否则就可能会引发内存安全问题。

试想一下，假如你在函数里头判断哪个字符串更长时，要返回的那个借用所指向的值被销毁了，而你却并不知情。那么，你把这个借用赋值给`a`时，你是不是就造成了一个悬垂引用？是不是就引发内存安全问题了？

出于谨慎，借用检查器就要求程序员必须显式告诉它形参、实参与返回值的生命周期关系。

明白原因之后，让我们为函数添加显式生命周期注解：

```rust
fn longest<'a>(first: &'a str, second: &'a str) -> &'a str {
    if first.len() > second.len() {
        first
    } else {
        second
    }
}
```

此时再调用函数，其模式匹配为：

```rust
// 'x和'y分别与其对应的形参中的'a对应，函数内部不再为x和y分配新的生命周期
longest(            'x  x,           'y  y )
                     |  |             |  |
longest<'a>(first: &'a str, second: &'a str)
```

这样，`longest`函数便同时得到了`x`和`y`的借用及其生命周期信息，借用检查器也就能正确分析它们之间的生命周期关系了。同时，由于返回值也声明了`'a`这个泛型参数，因此，它的生命周期也便能和`x`和`y`中的其中一个关联起来了。这样，借用检查器便能正确得出`a`的合法生命周期了。

有了以上分析，我们将示例代码修改成这样：

```rust
fn main() {
    let x = String::from("abc");
    let a;
    {
        let y = String::from("abcd");
        a = longest(&x, &y);
    }
    println!("{a}")
}

fn longest<'a>(first: &'a str, second: &'a str) -> &'a str {
    if first.len() > second.len() {
        first
    } else {
        second
    }
}
```

`y`的长度比`x`长，因此`a`会获得对`y`的借用。但由于`y`的存活时间比`a`短，违反了 Rust 借用规则，因此，编译出错。

解决方法如下：

```rust
fn main() {
    let x = String::from("abc");
    {
        let y = String::from("abcd");
        let a = longest(&x, &y);
        println!("{a}");
    }
}

fn longest<'a>(first: &'a str, second: &'a str) -> &'a str {
    if first.len() > second.len() {
        first
    } else {
        second
    }
}
```

这样，`a`的存活时间就不大于`y`了，符合 Rust 借用规则，编译通过，并打印出字符串`abcd`。

## 总结

我相信不少初学者在阅读函数生命周期的相关教程时一定会一头雾水。这是因为，在解释函数生命周期的概念时，这类教程既没有着重提及显式生命周期注解的本质，也没有强调函数形参本身也是一种模式匹配。

至于结构体生命周期等，这些就比较简单了，读者可以自行了解。

总而言之，生命周期虽然不是 Rust 独有概念，但它同样也是保证内存安全的一项重要措施。至于显式生命周期注解，它的作用是帮助借用检查器判断不同值之间的生命周期关系。牢记这一点就可以了。

不说了，老婆在喊我回家吃阮饭呢~润了。
