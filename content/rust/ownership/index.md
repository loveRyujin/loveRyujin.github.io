+++
date = '2025-11-06T23:44:21+08:00'
draft = false
title = '认识所有权'
tags = ["Rust"]
+++

## 概念
***所有权*** 是rust管理内存的一组规则，rust编译器会在编译期检查一系列所有权的规则，如果违反了对应规则就无法通过编译。

## 所有权规则

Rust 的所有权系统有三条基本规则：

1. **每个值都有一个所有者（owner）**
2. **值在任意时刻只能有一个所有者**
3. **当所有者离开作用域时，值将被丢弃**

## 变量作用域

作用域是一个变量在程序中有效的范围。让我们看一个简单的例子：

```rust
{                      // s 在这里无效，它尚未声明
    let s = "hello";   // 从此处起，s 是有效的
    
    // 使用 s
    println!("{}", s);
}                      // 此作用域已结束，s 不再有效
```

当变量 `s` 进入作用域时，它就是有效的。这一直持续到它离开作用域为止。

## 示例

### 1. 所有权转移（Move）

对于存储在堆上的数据，当我们将一个变量赋值给另一个变量时，所有权会发生转移：

```rust
let s1 = String::from("hello");
let s2 = s1;  // s1 的所有权转移给了 s2

// println!("{}", s1);  // 错误！s1 已经无效
println!("{}", s2);     // 正确，s2 拥有所有权
```

在这个例子中，`String` 类型的数据存储在堆上。当 `s1` 赋值给 `s2` 时，所有权转移了，`s1` 变得无效。这样做可以避免二次释放（double free）的问题。

### 2. 克隆（Clone）

如果我们确实需要深度复制堆上的数据，可以使用 `clone` 方法：

```rust
let s1 = String::from("hello");
let s2 = s1.clone();  // 深度复制

println!("s1 = {}, s2 = {}", s1, s2);  // 两者都有效
```

使用 `clone` 会在堆上创建一份新的数据副本，因此两个变量都拥有各自的数据。

### 3. 栈上数据的复制

对于存储在栈上的简单类型（如整数），赋值操作会自动复制值：

```rust
let x = 5;
let y = x;  // x 的值被复制给 y

println!("x = {}, y = {}", x, y);  // 两者都有效
```

这是因为像整数这样的类型实现了 `Copy` trait，它们的大小在编译时是已知的，完全存储在栈上，所以复制其实际的值是快速的。

### 4. 函数与所有权

将值传递给函数会发生所有权转移或复制，就像赋值一样：

```rust
fn main() {
    let s = String::from("hello");  // s 进入作用域
    
    takes_ownership(s);              // s 的值移动到函数里
    // println!("{}", s);            // 错误！s 已经无效
    
    let x = 5;                       // x 进入作用域
    
    makes_copy(x);                   // x 的值被复制到函数里
    println!("{}", x);               // 正确！x 仍然有效
}

fn takes_ownership(some_string: String) {
    println!("{}", some_string);
}  // some_string 离开作用域并被丢弃

fn makes_copy(some_integer: i32) {
    println!("{}", some_integer);
}  // some_integer 离开作用域，但不会有特殊操作
```

### 5. 返回值与所有权

函数返回值也可以转移所有权：

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership 将返回值移给 s1
    
    let s2 = String::from("hello");     // s2 进入作用域
    
    let s3 = takes_and_gives_back(s2);  // s2 被移动到函数中
                                         // 函数返回值移给 s3
    // println!("{}", s2);              // 错误！s2 已经无效
    println!("{}", s3);                 // 正确
}

fn gives_ownership() -> String {
    let some_string = String::from("yours");
    some_string  // 返回 some_string 并移出给调用的函数
}

fn takes_and_gives_back(a_string: String) -> String {
    a_string  // 返回 a_string 并移出给调用的函数
}
```

## 总结

所有权是 Rust 最独特的特性，它让 Rust 无需垃圾回收器就能保证内存安全。理解所有权的关键点：

- 每个值都有唯一的所有者
- 所有权可以转移（move）但不能共享
- 需要复制时使用 `clone`
- 简单类型实现了 `Copy` trait，会自动复制
- 函数调用会转移或复制所有权
- 当所有者离开作用域，值会被自动释放

但是像上面频繁地移动所有权未免显得有些繁琐，如果每个变量都这样处理会增加很多心智负担，所以引申出后面要介绍的概念：***引用***