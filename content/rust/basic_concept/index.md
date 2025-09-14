+++
date = '2025-09-14T12:53:10+08:00'
draft = false
title = 'Basic_concept'
tags = ["Rust"]
+++

## 变量
rust中使用**let**定义变量，变量类型可以不显示声明，编译器会自动推导。
```rust
let x = 5;
```
变量默认不可变，当对一个不可变变量重新赋值时，编译器会报错。
```rust
let x = 5;
x = 6;
# error[E0384]: cannot assign twice to immutable variable `x`

```
如果想让变量可变，需要声明时带上**mut**。
```rust
let mut x = 5;
x = 6;
```

rust存在变量遮蔽，指的是同一个作用域下，使用let关键字重新定义一个与之前变量同名的新变量，这个新定义的变量会覆盖掉之前的定义的旧变量，后续只能访问这个新变量，直至新变量的作用域结束。

这里可能会疑惑，貌似和可变性有点类似，我们对比一下：
|特性|变量遮蔽|可变性|
|:---|:---|:---|
|**关键字**|let|mut|
|**创建**|创建一个全新的变量|改变同一个变量的值|
|**类型**|可以改变变量的类型|不能改变变量的类型|
|**作用**|隐藏旧变量，使用新变量|允许修改当前变量的值|

变量遮蔽给我们带来一些便利性，比如当我们需要对一个变量进行一系列类型转换的时候，可以重复定义一个变量名，而不用针对每个类型都定义一个变量名。

---

## 数据类型
rust中数据类型分为两类：**标量类型**和**复合类型**。

### 标量类型
标量类型表示单个值，包括：
- 整数
- 浮点数
- 布尔值
- 字符

### 复合类型
复合类型可以将多个值组合成一个类型，原生复合类型包括：
- 元组
- 数组

---

## 函数
rust使用**fn**关键字定义函数。
```rust
fn print_hello() {
    println!("Hello, world!");
}
```
函数可以有参数，参数必须指定类型。
```rust
# 使用{}代表占位符
fn print_hello(name: String) {
    println!("Hello, {}!", name);
}
```
函数可以有返回值，返回值需要指定类型。
```rust
fn print_hello(name: String) -> String {
    return format!("Hello, {}!", name);
}
```
使用**reture**关键字可以提前返回，大部分函数隐式返回最后的表达式。

函数体通常由一系列语句和表达式组成。
- **语句**是执行一些操作但不返回值的指令。
- **表达式**是计算并返回值的代码片段。

---

## 流程控制
和其它语言类似，rust也提供了常见的流程控制语句，包括：
- if/else
- loop
- while
- for
- match

### if/else
```rust
let x = 5;
if x > 0 {
    println!("x is positive");
} else {
    println!("x is negative");
}

```
**if**后边必须是一个布尔表达式。

### loop
**loop**会一遍遍执行，直到遇到**break**语句。
```rust
let mut x = 0;
loop {
    x += 1;
    println!("x is {}", x);
}
```
**loop**循环还可以返回值，可以使用**break**语句返回值。
```rust
let mut x = 0;
let result = loop {
    x += 1;
    if x == 10 {
        break x;
    }
};
println!("result is {}", result);
```

## while
**while**循环会一直执行，直到条件不成立。
```rust
let mut x = 10;
while x > 0 {
    println!("x is {}", x);
    x -= 1;
}
```

### for
**for**循环会遍历一个集合中的所有元素。
```rust
let arr = [1, 2, 3, 4, 5];
for x in arr {
    println!("x is {}", x);
}
```

### match
**match**用于模式匹配，类似其它语言的**switch**语句。

**match**必须覆盖所有可能的值，编译器会强制要求处理所有可能的情况。

同时**match**不仅可以匹配简单的字面量，还可以对复杂的数据结构进行模式匹配。

匹配字面量：
```rust
let x = 2;
match x {
    1 => println!("x is 1"),
    2 => println!("x is 2"),
    _ => println!("x is other"),
}
```

匹配元组：
```rust
let point = (0, 7);
match point {
    (0, y) => println!("在 x 轴上, y = {}", y), // 匹配 x 为 0 的情况并绑定 y 的值
    (x, 0) => println!("在 y 轴上, x = {}", x),
    (x, y) => println!("不在轴上, x={}, y={}", x, y),
}
```

使用 **|** 可以匹配多个模式：
```rust
let x = 5;
match x {
    1 | 2 => println!("1 或 2"),
    3..=5 => println!("3 到 5 之间"), // 还可以匹配范围
    _ => println!("其他"),
}
```

---

## 注释
rust中主要包括行注释、文档注释和块注释。

### 行注释
rust使用 **//** 注释单行。

### 文档注释
rust使用 **///** 或者 **//！** 注释单行。

### 块注释
rust使用 **/\*...\*/** 注释多行。

