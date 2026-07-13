---
title: "Structs, Enums & Pattern Matching"
subtitle: ""
date: 2026-07-09T09:00:00+07:00
lastmod: 2026-07-09T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Build your own data types with structs and enums, and handle every case safely with pattern matching"
license: ""
images: []
tags: ["Rust", "Tutorial"]
categories: ["Rust"]
featuredImage: "featured-image.jpeg"
featuredImagePreview: "featured-image.jpeg"
lightgallery: true
---

Structs and enums are the building blocks for creating custom data types in Rust. They allow you to model the data of your domain with precision and safety. In the rs-wsProxy project, you'll see structs like `Args` for command-line configuration and enums like `VerifyResult` for representing the outcome of a verification process. This tutorial builds on the concepts of ownership and borrowing covered in [Part 2](/posts/rust/rust-ownership-borrowing/) and prepares you for [Part 4](/posts/rust/rust-collections-errors/) on collections, collections and error handling.

<!--more-->

## Structs

A struct, or structure, is a custom data type that lets you group together related values of different types. Think of it as a blueprint for creating instances that share the same structure.

### Named-Field Structs

The most common kind of struct is the named-field struct. Each field has a name and a type. For example, we can model a `User` with a username and email:

```rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
```

To create an instance of this struct, we specify values for each field:

```rust
let user1 = User {
    username: String::from("kawin"),
    email: String::from("kawin@example.com"),
    sign_in_count: 1,
    active: true,
};
```

Rust provides a convenient shorthand when the variable names match the field names. If we have variables with the same names as the fields, we can use the field init shorthand:

```rust
let username = String::from("kawin");
let email = String::from("kawin@example.com");
let sign_in_count = 1;
let active = true;

let user1 = User {
    username,
    email,
    sign_in_count,
    active,
};
```

### Tuple Structs

Sometimes you want a struct that doesn't need named fields but just wants to give a tuple a distinct type. This is where tuple structs come in. They look like tuples but have a struct name.

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

let black = Color(0, 0, 0);
let origin = Point(0, 0, 0);
```

Even though `Color` and `Point` both contain three `i32` values, they are different types and cannot be used interchangeably.

### Unit Structs

A unit struct is a struct without any fields. It's useful when you need a type that carries no data but can be used as a marker or for implementing traits.

```rust
struct AlwaysEqual;

let subject = AlwaysEqual;
```

### Accessing Fields

To access a field of a struct instance, we use the dot notation:

```rust
let username = user1.username;
let email = user1.email;
```

If the instance is mutable, we can change a field's value:

```rust
let mut user1 = User {
    username: String::from("kawin"),
    email: String::from("kawin@example.com"),
    sign_in_count: 1,
    active: true,
};

user1.email = String::from("another@example.com");
```

## Methods with `impl`

Methods are functions defined within the context of a struct (or enum). They are defined inside an `impl` block. The first parameter of a method is often `self`, which represents the instance the method is called on.

### Defining Methods

Let's add some functionality to our `User` struct. We'll create a method to check if the user is active and another to increment the sign-in count.

```rust
impl User {
    fn is_active(&self) -> bool {
        self.active
    }

    fn increment_sign_in_count(&mut self) {
        self.sign_in_count += 1;
    }

    // This is an associated function because it doesn't take `self`
    fn new(username: String, email: String) -> User {
        User {
            username,
            email,
            sign_in_count: 0,
            active: true,
        }
    }
}
```

In the `is_active` method, we use `&self` because we only need to read the instance. In `increment_sign_in_count`, we use `&mut self` because we need to modify the instance. The `new` function is an associated function (often used as a constructor) because it doesn't take `self`; it's called with `User::new(...)`.

### Using Methods

Here's how we use these methods:

```rust
let mut user = User::new(String::from("kawin"), String::from("kawin@example.com"));

println!("Is active? {}", user.is_active()); // Prints: Is active? true

user.increment_sign_in_count();
println!("Sign in count: {}", user.sign_in_count); // Prints: Sign in count: 1
```

### Multiple `impl` Blocks

You can define multiple `impl` blocks for the same struct. This is useful for organizing methods by functionality or for separating associated functions from methods.

```rust
impl User {
    // Methods related to user state
    fn is_active(&self) -> bool {
        self.active
    }
}

impl User {
    // Methods related to user activity
    fn increment_sign_in_count(&mut self) {
        self.sign_in_count += 1;
    }

    fn reset_sign_in_count(&mut self) {
        self.sign_in_count = 0;
    }
}
```

This is equivalent to having all methods in a single `impl` block.

### Example: Rectangle

Let's look at a more comprehensive example with a `Rectangle` struct that has methods for calculating area and scaling.

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }

    fn scale(&mut self, factor: u32) {
        self.width *= factor;
        self.height *= factor;
    }

    // Associated function to create a square
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}

fn main() {
    let mut rect = Rectangle {
        width: 30,
        height: 50,
    };

    println!("Area: {}", rect.area()); // 1500

    rect.scale(2);
    println!("Scaled area: {}", rect.area()); // 6000

    let sq = Rectangle::square(10);
    println!("Square area: {}", sq.area()); // 100
}
```

## Enums

Enums, short for enumerations, allow you to define a type by listing its possible variants. Each variant can hold different types and amounts of data.

### Simple Enums

A simple enum lists variants that don't hold any additional data. For example, a `Direction` enum for moving in a grid:

```rust
enum Direction {
    Up,
    Down,
    Left,
    Right,
}

let north = Direction::Up;
let east = Direction::Right;
```

### Enums with Data

Enums become much more powerful when variants can hold data. This is similar to unions or variant types in other languages, but with Rust's safety guarantees.

Consider an enum for IP addresses, which can be either IPv4 or IPv6:

```rust
enum IpAddr {
    V4(String),
    V6(String),
}

let home = IpAddr::V4(String::from("127.0.0.1"));
let loopback = IpAddr::V6(String::from("::1"));
```

Each variant holds a `String`, but we could also hold other types. For instance, we might want to store IPv4 addresses as four `u8` values and IPv6 as six `u16` values:

```rust
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

let home = IpAddr::V4(127, 0, 0, 1);
let loopback = IpAddr::V6(String::from("::1"));
```

### Complex Enums with Multiple Data Types

An enum can have variants that hold different types and amounts of data. This is where enums really shine as algebraic data types.

Let's define an enum for different kinds of messages we might receive in a network application:

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

let msg1 = Message::Quit;
let msg2 = Message::Move { x: 10, y: 20 };
let msg3 = Message::Write(String::from("hello"));
let msg4 = Message::ChangeColor(255, 0, 255);
```

Here:
- `Quit` has no data.
- `Move` holds a struct-like block with `x` and `y` coordinates.
- `Write` holds a `String`.
- `ChangeColor` holds three `i32` values for red, green, and blue.

This ability to model data with precise variation is why enums are so useful in Rust.

### Why Enums Are Powerful

Enums in Rust are algebraic data types, meaning they can represent data that is one of several variants, each with its own data. This is in contrast to structs, which represent data that is a combination of all fields. Enums let you express ideas like "this value is either A with some data, or B with other data, or C with no data" in a type-safe way.

## The `Option<T>` Enum

One of the most important enums in the Rust standard library is `Option<T>`. It represents the presence or absence of a value, eliminating the need for null pointers.

### Definition

The `Option<T>` enum is defined as follows:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

- `Some(T)` indicates that a value is present and holds the value of type `T`.
- `None` indicates that there is no value.

### Using `Option<T>`

Let's look at a function that searches for a character in a string and returns its index, or `None` if the character isn't found:

```rust
fn find_char(s: &str, c: char) -> Option<usize> {
    for (index, &current_char) in s.as_bytes().iter().enumerate() {
        if current_char as char == c {
            return Some(index);
        }
    }
    None
}

fn main() {
    let s = String::from("hello world");

    match find_char(&s, 'o') {
        Some(index) => println!("Found 'o' at index {}", index),
        None => println!("Character 'o' not found"),
    }

    match find_char(&s, 'z') {
        Some(index) => println!("Found 'z' at index {}", index),
        None => println!("Character 'z' not found"), // This will print
    }
}
```

### Why Safer Than Null

In languages with null, accessing a null pointer often leads to runtime errors. With `Option<T>`, you must explicitly handle both the `Some` and `None` cases. The compiler enforces this handling, preventing accidental null dereferences.

For example, if we tried to use the result of `find_char` without checking, we'd get a compile-time error:

```rust
// This won't compile because we're trying to use an Option<usize> as a usize
let index = find_char(&s, 'o');
println!("Index: {}", index); // Error: expected usize, found enum Option
```

We must use a `match` statement or methods like `unwrap()` (which panics on `None`) or `expect()` to handle the `Option`.

## Pattern Matching

Pattern matching in Rust is done with the `match` keyword. It allows you to compare a value against a series of patterns and execute code based on which pattern matches. Patterns can be literal values, variable names, wildcards, and more.

### Exhaustive Checking

One of the key features of `match` is that it is exhaustive. You must cover every possible case, or the compiler will complain. This ensures that you don't accidentally miss a variant.

### Matching `Option<T>`

Let's revisit the `find_char` example and see how we handle the `Option` with `match`:

```rust
fn find_char(s: &str, c: char) -> Option<usize> {
    for (index, &current_char) in s.as_bytes().iter().enumerate() {
        if current_char as char == c {
            return Some(index);
        }
    }
    None
}

fn main() {
    let s = String::from("hello world");

    let result = find_char(&s, 'o');
    match result {
        Some(index) => println!("Found 'o' at index {}", index),
        None => println!("Character 'o' not found"),
    }
}
```

In this `match`, we have two arms: one for `Some(index)` and one for `None`. The variable `index` is bound to the value inside `Some`.

### Patterns with Bindings

You can bind parts of a pattern to variables. This is useful for extracting data from enum variants.

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn process_message(msg: Message) {
    match msg {
        Message::Quit => {
            println!("Quit command received");
        }
        Message::Move { x, y } => {
            println!("Move to x: {}, y: {}", x, y);
        }
        Message::Write(text) => {
            println!("Text message: {}", text);
        }
        Message::ChangeColor(r, g, b) => {
            println!(
                "Change color to red: {}, green: {}, blue: {}",
                r, g, b
            );
        }
    }
}

fn main() {
    let msg1 = Message::Quit;
    let msg2 = Message::Move { x: 10, y: 20 };
    let msg3 = Message::Write(String::from("hello"));
    let msg4 = Message::ChangeColor(255, 0, 255);

    process_message(msg1);
    process_message(msg2);
    process_message(msg3);
    process_message(msg4);
}
```

### The `_` Wildcard

The underscore (`_`) is a catch-all pattern that matches any value. It's useful when you want to ignore a value or handle all remaining cases with a single arm.

```rust
fn main() {
    let some_u8_value = 0u8;

    match some_u8_value {
        1 => println!("one"),
        3 => println!("three"),
        5 => println!("five"),
        7 => println!("seven"),
        _ => println!("anything else"), // This will print for 0
    }
}
```

### `if let` for Single Case

When you only care about one variant and want to ignore the others, `if let` provides a more concise syntax than `match`.

```rust
fn main() {
    let some_option = Some(5);

    if let Some(i) = some_option {
        println!("{}", i); // Prints: 5
    }
}

// This is equivalent to:
//
// match some_option {
//     Some(i) => {
//         println!("{}", i);
//     }
//     _ => (),
// }
```

### `while let` for Loops

Similarly, `while let` allows you to loop as long as a pattern matches.

```rust
fn main() {
    let mut stack = Vec::new();

    stack.push(1);
    stack.push(2);
    stack.push(3);

    while let Some(top) = stack.pop() {
        println!("{}", top); // Prints 3, 2, 1
    }
}
```

## The `Result<T, E>` Enum (Intro)

Just like `Option<T>` handles the presence or absence of a value, `Result<T, E>` handles success or failure. It's used for operations that can fail, such as file I/O or parsing.

### Definition

The `Result<T, E>` enum is defined as:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

- `Ok(T)` indicates that the operation succeeded and holds the result of type `T`.
- `Err(E)` indicates that the operation failed and holds an error of type `E`.

### Brief Example

Here's a simple example that tries to parse a string into an integer:

```rust
use std::num::ParseIntError;

fn double_number(number_str: &str) -> Result<i32, ParseIntError> {
    match number_str.parse::<i32>() {
        Ok(n) => Ok(2 * n),
        Err(e) => Err(e),
    }
}

fn main() {
    match double_number("10") {
        Ok(n) => println!("Double: {}", n),
        Err(e) => println!("Error: {}", e),
    }

    match double_number("hello") {
        Ok(n) => println!("Double: {}", n),
        Err(e) => println!("Error: {}", e),
    }
}
```

In the next part, we'll dive deeper into `Result<T, E>`, combinators like `map` and `and_then`, and the `?` operator for error propagation.

## Connection to rs-wsProxy

Let's see how these concepts are used in the rs-wsProxy project. The `Args` struct holds command-line configuration, and the `VerifyResult` enum represents the outcome of verifying a connection request.

```rust
pub struct Args {
    pub config: String,
    pub verbose: bool,
}

pub enum VerifyResult {
    Accepted(String),
    Rejected(RejectReason),
}

pub struct AppState {
    pub allowed_servers: Option<Vec<String>>,
    pub redirects: HashMap<String, String>,
    pub default_target: Option<String>,
}
```

### Understanding `Option` in `AppState`

The `AppState` struct uses `Option` in two fields:
- `allowed_servers: Option<Vec<String>>`
- `default_target: Option<String>`

These options distinguish between three states:
1. `None`: The field is not set. For `allowed_servers`, this means the proxy is open (allows connections to any server). For `default_target`, it means there is no fallback server.
2. `Some(Vec<String>)`: A list of allowed servers. An empty vector `Some(vec![])` means no servers are allowed (deny all).
3. `Some(String)`: A specific default target server.

This design makes the semantics explicit and prevents confusion between an empty list and an unset value.

### Matching on `VerifyResult`

When handling a connection request, we match on `VerifyResult` to decide what to do:

```rust
fn handle_verify(result: VerifyResult) {
    match result {
        VerifyResult::Accepted(server) => {
            println!("Connection accepted to {}", server);
            // Proceed with establishing the connection
        }
        VerifyResult::Rejected(reason) => {
            println!("Connection rejected: {:?}", reason);
            // Send an error response or close the connection
        }
    }
}
```

This match is exhaustive: we must handle both `Accepted` and `Rejected` variants, ensuring we don't miss any possible outcome.

## Summary

In this tutorial, we've explored how to define and use structs and enums to create custom data types in Rust. We've seen how to attach methods to structs with `impl` blocks, and how enums like `Option<T>` and `Result<T, E>` provide powerful ways to handle absence and error cases safely. Pattern matching with `match`, `if let`, and `while let` allows us to handle each variant of an enum clearly and exhaustively.

These concepts are foundational for writing idiomatic Rust code and are heavily used in real-world projects like rs-wsProxy. By modeling data with structs and enums, and handling all cases with pattern matching, you can write code that is both expressive and robust.

### Navigation

← Previous: [Ownership, Borrowing & Lifetimes](/posts/rust/rust-ownership-borrowing/)
Next: [Collections, Iterators & Error Handling](/posts/rust/rust-collections-errors/)