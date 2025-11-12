+++
date = '2025-11-12T23:21:30+08:00'
draft = false
title = '引用与借用'
tags = ["Rust"]
series = ["Rust入门教程"]
series_order = 3
+++

## 概念

***引用***（Reference）类似指针，是一个地址，通过引用可以访问该地址下的数据。但与指针不同的是，引用在 Rust 中是安全的，编译器会保证引用总是指向有效的值。

当我们使用引用时，我们称之为***借用***（Borrowing）。就像在现实生活中，你可以借用别人的东西，但你不拥有它，使用完后需要归还。在 Rust 中，引用允许你使用值但不获取其所有权。

## 借用规则

Rust 的借用系统有两条核心规则，这些规则在编译时强制执行：

1. **在任意给定时间，要么只能有一个可变引用，要么只能有任意数量的不可变引用**
2. **引用必须总是有效的**（不能出现悬垂引用）

这些规则确保了在编译时就能防止数据竞争。

## 不可变引用

使用 `&` 符号创建一个不可变引用，它允许你读取数据但不能修改：

```rust
fn main() {
    let s1 = String::from("hello");
    
    let len = calculate_length(&s1);  // 传递引用，不转移所有权
    
    println!("The length of '{}' is {}.", s1, len);  // s1 仍然有效
}

fn calculate_length(s: &String) -> usize {
    s.len()
}  // s 离开作用域，但因为它不拥有所有权，所以不会释放内存
```

在这个例子中，`&s1` 创建了一个指向 `s1` 的引用，但不拥有它。因为没有所有权转移，所以在调用函数后 `s1` 仍然有效。

### 多个不可变引用

你可以同时拥有多个不可变引用，因为只读操作不会造成数据竞争：

```rust
fn main() {
    let s = String::from("hello");
    
    let r1 = &s;  // 没问题
    let r2 = &s;  // 没问题
    let r3 = &s;  // 没问题
    
    println!("{}, {}, {}", r1, r2, r3);  // 都可以使用
}
```

### 不可变引用的限制

通过不可变引用无法修改数据：

```rust
fn main() {
    let s = String::from("hello");
    
    change(&s);  // 编译错误！
}

fn change(some_string: &String) {
    some_string.push_str(", world");  // 错误：不能修改借用的内容
}
```

## 可变引用

如果需要修改借用的值，可以使用 `&mut` 创建可变引用：

```rust
fn main() {
    let mut s = String::from("hello");  // s 必须是可变的
    
    change(&mut s);  // 传递可变引用
    
    println!("{}", s);  // 输出：hello, world
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

### 可变引用的限制

可变引用有一个重要的限制：**在同一作用域内，对同一数据只能有一个可变引用**。

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &mut s;
    let r2 = &mut s;  // 错误！不能同时有两个可变引用
    
    println!("{}, {}", r1, r2);
}
```

这个限制的好处是可以在编译时防止数据竞争。数据竞争会导致以下行为：

- 两个或更多指针同时访问同一数据
- 至少有一个指针被用来写入数据
- 没有同步数据访问的机制

### 可变引用与不可变引用不能共存

你也不能在拥有不可变引用的同时拥有可变引用：

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &s;      // 没问题
    let r2 = &s;      // 没问题
    let r3 = &mut s;  // 错误！不能在有不可变引用时创建可变引用
    
    println!("{}, {}, {}", r1, r2, r3);
}
```

但是，如果不可变引用的作用域已经结束，就可以创建可变引用：

```rust
fn main() {
    let mut s = String::from("hello");
    
    let r1 = &s;      // 没问题
    let r2 = &s;      // 没问题
    println!("{} and {}", r1, r2);
    // r1 和 r2 在这之后不再使用
    
    let r3 = &mut s;  // 没问题！
    println!("{}", r3);
}
```

这段代码能够编译，因为 `r1` 和 `r2` 的作用域在 `println!` 之后就结束了，这个位置称为引用的***非词法作用域生命周期***（Non-Lexical Lifetimes, NLL）。

## 悬垂引用

悬垂引用（Dangling Reference）是指引用指向的内存已经被释放。在 Rust 中，编译器会保证引用永远不会是悬垂的：

```rust
fn main() {
    let reference_to_nothing = dangle();  // 编译错误！
}

fn dangle() -> &String {  // 返回一个 String 的引用
    let s = String::from("hello");
    
    &s  // 返回 s 的引用
}  // s 离开作用域并被丢弃，其内存被释放
   // 危险！引用指向了无效的内存
```

编译器会阻止这种情况发生。正确的做法是直接返回值，转移所有权：

```rust
fn no_dangle() -> String {
    let s = String::from("hello");
    s  // 所有权被转移出去，没有值被释放
}
```

## 引用作为函数参数

使用引用作为函数参数是一种常见的模式，可以避免所有权的转移：

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = String::from("world");
    
    let result = first_word(&s1);
    println!("The first word is: {}", result);
    
    // s1 和 s2 仍然有效，可以继续使用
    println!("{} {}", s1, s2);
}

fn first_word(s: &String) -> &str {
    let bytes = s.as_bytes();
    
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    
    &s[..]
}
```

## 引用的实际应用场景

### 1. 避免不必要的克隆

```rust
// 不好的做法：克隆数据
fn process_data(data: String) -> usize {
    data.len()
}

fn main() {
    let data = String::from("hello world");
    let len = process_data(data.clone());  // 克隆开销大
    println!("{}", data);
}

// 好的做法：使用引用
fn process_data_ref(data: &String) -> usize {
    data.len()
}

fn main() {
    let data = String::from("hello world");
    let len = process_data_ref(&data);  // 只传递引用
    println!("{}", data);
}
```

### 2. 在循环中使用引用

```rust
fn main() {
    let strings = vec![
        String::from("hello"),
        String::from("world"),
        String::from("rust"),
    ];
    
    // 使用引用遍历，避免所有权转移
    for s in &strings {
        println!("{}", s);
    }
    
    // strings 仍然有效
    println!("Total strings: {}", strings.len());
}
```

### 3. 修改集合中的元素

```rust
fn main() {
    let mut numbers = vec![1, 2, 3, 4, 5];
    
    // 使用可变引用修改元素
    for num in &mut numbers {
        *num *= 2;  // 解引用并修改
    }
    
    println!("{:?}", numbers);  // 输出：[2, 4, 6, 8, 10]
}
```

## 总结

引用和借用是 Rust 中非常重要的概念，它们让你能够在不转移所有权的情况下使用数据：

- **不可变引用** (`&T`)：允许读取数据，可以同时存在多个
- **可变引用** (`&mut T`)：允许修改数据，但在同一作用域内只能有一个
- **借用规则**：要么多个不可变引用，要么一个可变引用，不能同时存在
- **引用必须有效**：编译器会防止悬垂引用
- **引用不拥有所有权**：引用离开作用域时不会释放数据

