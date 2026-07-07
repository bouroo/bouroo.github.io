---
title: "Getting Started with Rust"
subtitle: ""
date: 2026-07-07T09:00:00+07:00
lastmod: 2026-07-07T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Install Rust, learn Cargo and basic syntax — the first step toward building a WebSocket-to-TCP proxy"
license: ""
images: []
tags: ["Rust", "Tutorial"]
categories: ["Rust"]
featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"
lightgallery: true
---

# Getting Started with Rust

This series started when I saw a Facebook post from [rayrag.com](https://rayrag.com/) showing that you can play **Ragnarok Online (RO)** right in your web browser — connecting through WebSocket to an emulator game server. That got me curious about how it works under the hood, and I wanted to build my own WebSocket-to-TCP proxy using a language that's fast and safe. That's how I came back to revisit Rust and created the [rs-wsProxy](https://github.com/bouroo/rs-wsProxy) project.

<!--more-->

Welcome to the first part of our 8-part series "Learning Rust — Building a WebSocket-to-TCP Proxy". In this series, we'll revisit Rust fundamentals and build a production-ready WebSocket-to-TCP proxy that works with [roBrowser](https://github.com/vthibault/roBrowser).

## Why Rust?

Rust offers a unique combination of performance, safety, and concurrency without garbage collection. It gives you control over low-level details while preventing entire classes of bugs at compile time. This makes it perfect for building network applications like our WebSocket-to-TCP proxy where performance and reliability are critical.

## Installing Rust

The recommended way to install Rust is through `rustup`, which manages Rust versions and associated tools.

Open your terminal and run:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

This script will download and install `rustup`, which in turn installs:
- `rustc` - the Rust compiler
- `cargo` - Rust's package manager and build tool
- `rust-std` - the standard library
- `rust-docs` - documentation

After installation, you need to source the environment:

```bash
source $HOME/.cargo/env
```

Verify the installation:

```bash
rustc --version
cargo --version
```

You should see output similar to:
```
rustc 1.70.0 (90c541806 2023-05-31)
cargo 1.70.0 (aba3780b2 2023-05-15)
```

To update Rust to the latest stable version later, simply run:

```bash
rustup update
```

## Hello, Cargo

Cargo is Rust's build system and package manager. It handles downloading dependencies, compiling your code, and creating distributable packages.

Let's create a new project:

```bash
cargo new hello_rust
cd hello_rust
```

This creates a directory structure:
```
hello_rust/
├── Cargo.toml
└── src/
    └── main.rs
```

`Cargo.toml` is the manifest file where you configure your project and dependencies. `src/main.rs` contains the source code.

Let's look at the generated `main.rs`:

```rust
fn main() {
    println!("Hello, world!");
}
```

Run the program:

```bash
cargo run
```

Output:
```
Hello, world!
```

For optimized builds (suitable for release), use:

```bash
cargo build --release
```

The compiled binary will be in `target/release/hello_rust`.

Let's examine `Cargo.toml`:

```toml
[package]
name = "hello_rust"
version = "0.1.0"
edition = "2021"

[dependencies]
```

Key sections:
- `[package]`: Metadata about your project
- `[dependencies]`: External crates your project depends on

The `edition = "2021"` specifies we're using Rust 2021 edition, which we'll use throughout this tutorial.

## Variables and Mutability

In Rust, variables are immutable by default. This helps prevent accidental changes and makes code easier to reason about.

```rust
let x = 5; // immutable
// x = 6; // This would cause a compile-time error
```

To make a variable mutable, use `mut`:

```rust
let mut y = 5;
y = 6; // This is allowed
println!("y = {}", y); // Prints: y = 6
```

### Shadowing

You can declare a new variable with the same name as a previous variable. This is called shadowing.

```rust
let z = 5;
let z = z + 1; // New variable z shadows the previous z
let z = z * 2;
println!("z = {}", z); // Prints: z = 12
```

Shadowing is different from mutability because it creates a new variable, allowing you to change the type if needed.

### Constants

Constants are similar to immutable variables but are declared with `const` and must be annotated with a type. They're valid for the entire time a program runs.

```rust
const MAX_POINTS: u32 = 100_000;
```

Constants can be declared in any scope, including global scope, and must be set to a constant expression.

## Data Types

Rust is statically typed, meaning types must be known at compile time. However, the compiler can often infer types.

### Scalar Types

Scalar types represent a single value. Rust has four primary scalar types:

#### Integers
Integers come in signed and unsigned variants, with different sizes:

| Length | Signed | Unsigned |
|--------|--------|----------|
| 8-bit  | i8     | u8       |
| 16-bit | i16    | u16      |
| 32-bit | i32    | u32      |
| 64-bit | i64    | u64      |
| 128-bit| i128   | u128     |
| arch   | isize  | usize    |

`isize` and `usize` depend on the architecture of the computer (64-bit or 32-bit).

Example:
```rust
let a: i32 = 42;
let b: u64 = 1_000_000_000;
```

#### Floating-Point Numbers
Rust has two primitive types for floating-point numbers:

```rust
let x = 2.0; // f64 by default
let y: f32 = 3.0; // f32
```

#### Boolean
The `bool` type has two possible values: `true` and `false`.

```rust
let is_rust_fun = true;
let is_java_fun = false;
```

#### Character
The `char` type represents a Unicode scalar value, meaning it can represent much more than just ASCII.

```rust
let c = 'z';
let z = 'ℤ';
let heart_eyed_cat = '😻';
```

Note: Char literals are specified with single quotes, unlike string literals which use double quotes.

### Compound Types

Compound types can group multiple values into one type. Rust has two primitive compound types: tuples and arrays.

#### Tuples
Tuples group together a fixed number of values with different types.

```rust
let tup: (i32, f64, u8) = (500, 6.4, 1);
```

To get the individual values out of a tuple, we can use pattern matching to destructure:

```rust
let (x, y, z) = tup;
println!("The value of y is: {}", y); // Prints: 6.4
```

Alternatively, we can access elements directly with a dot followed by the index:

```rust
let five_hundred = tup.0;
let six_point_four = tup.1;
let one = tup.2;
```

#### Arrays
Arrays in Rust have a fixed length, and every element must have the same type.

```rust
let a = [1, 2, 3, 4, 5];
```

Arrays are useful when you want your data allocated on the stack rather than the heap, or when you want to ensure you always have a fixed number of elements.

You can also initialize an array with the same value for each element:

```rust
let b = [3; 5]; // Creates [3, 3, 3, 3, 3]
```

To access array elements, use indexing:

```rust
let first = a[0];
let second = a[1];
```

Note: Accessing an index beyond the array's length will cause a runtime panic.

## Functions

Functions are the building blocks of Rust code. We've already seen the `main` function, which is the entry point of every executable program.

Let's look at a simple function:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

This function takes two `i32` parameters and returns their sum as an `i32`.

### Expressions vs Statements

Rust is an expression-based language, which means most things are expressions rather than statements.

- **Statements** perform actions but don't return a value.
- **Expressions** evaluate to a resulting value.

In the `add` function above, `a + b` is an expression that evaluates to the sum. The entire function body is an expression, and its value is returned implicitly (note the lack of semicolon).

If we add a semicolon, it becomes a statement:

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b; // This is a statement, returns ()
}
```

This would cause a compile error because the function is expected to return an `i32` but returns `()` (unit type) instead.

### Early Return

You can return early from a function using the `return` keyword:

```rust
fn foo(x: i32) -> i32 {
    if x < 0 {
        return -1; // Early return
    }
    x * x
}
```

However, Rust encourages using expressions and match statements for control flow rather than early returns when possible.

## Control Flow

Control flow determines the order in which code executes based on conditions.

### if/else

```rust
let number = 6;

if number % 4 == 0 {
    println!("number is divisible by 4");
} else if number % 3 == 0 {
    println!("number is divisible by 3");
} else if number % 2 == 0 {
    println!("number is divisible by 2");
} else {
    println!("number is not divisible by 4, 3, or 2");
}
```

### if as Expression

Because `if` is an expression, we can use it on the right-hand side of a let statement:

```rust
let condition = true;
let number = if condition { 5 } else { 6 };

println!("The value of number is: {}", number); // Prints: 5
```

Note that all arms of the if must return the same type.

### loop

Rust provides a `loop` keyword to execute a block of code repeatedly until you explicitly tell it to stop.

```rust
let mut counter = 0;

let result = loop {
    counter += 1;

    if counter == 10 {
        break counter * 2; // We can return a value from break
    }
};

println!("The result is {}", result); // Prints: 20
```

### while

A `while` loop is useful when you want to loop as long as a condition holds.

```rust
let mut number = 3;

while number != 0 {
    println!("{}!", number);
    number -= 1;
}

println!("LIFTOFF!!!");
```

### for with Ranges

The `for` loop is commonly used to iterate over a collection. We can use ranges to generate a sequence of numbers.

```rust
// Range expression: start..end (exclusive of end)
for number in 0..5 {
    println!("{}!", number);
}
// Prints: 0! 1! 2! 3! 4!

// Inclusive range: start..=end
for number in 1..=5 {
    println!("{}!", number);
}
// Prints: 1! 2! 3! 4! 5!
```

## Comments

Comments are ignored by the compiler but help humans understand the code.

### Line Comments

Line comments start with `//` and continue to the end of the line.

```rust
// This is a line comment
let x = 5; // This is also a line comment
```

### Block Comments

Block comments start with `/*` and end with `*/`. They can span multiple lines.

```rust
/* This is a block comment
   that spans multiple lines */
let y = 6;
```

Block comments can be nested:

```rust
/* This is a block comment
   /* with a nested block comment */
   and back to the outer block */
let z = 7;
```

### Documentation Comments

Documentation comments generate HTML documentation. They use three slashes (`///`) or an exclamation mark followed by two slashes (`//!`).

```rust
/// Adds one to the number given.
///
/// # Examples
///
/// ```
/// let arg = 5;
/// let answer = my_crate::add_one(arg);
/// assert_eq!(6, answer);
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

Module documentation comments (`//!`) are placed at the top of a file to document the entire module:

```rust
//! # My Crate
//!
//! `my_crate` is a collection of utilities to make performing certain
//! calculations more convenient.
```

## Summary

In this first part of our Rust series, we've covered:
- Why Rust is a great choice for systems programming
- How to install and update Rust using rustup
- Creating and running your first Cargo project
- Variables, mutability, shadowing, and constants
- Rust's data types including scalars (integers, floats, booleans, chars) and compounds (tuples, arrays)
- Writing functions, understanding expressions vs statements, and early returns
- Control flow with if/else, loops, and for loops with ranges
- Different types of comments including documentation comments

You now have enough knowledge to write basic Rust programs and understand the syntax. In the next part, we'll dive into Rust's ownership system, which is what makes Rust unique in providing memory safety without garbage collection.

**Next: [Ownership, Borrowing & Lifetimes](/posts/rust/rust-ownership-borrowing/)**

Continue your Rust journey by learning about ownership, the cornerstone of Rust's safety guarantees.