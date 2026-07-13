---
title: "Collections, Iterators & Error Handling"
subtitle: ""
date: 2026-07-10T09:00:00+07:00
lastmod: 2026-07-10T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Learn Vec, HashMap, iterators, and error handling with Result and the ? operator"
license: ""
images: []
tags: ["Rust", "Tutorial"]
categories: ["Rust"]
featuredImage: "featured-image.jpeg"
featuredImagePreview: "featured-image.jpeg"
lightgallery: true
---

Collections store multiple values, iterators process them, and error handling keeps code safe. These three concepts form the backbone of practical Rust programming. In Part 3 of our Rust series, we covered structs, enums, and pattern matching. Now we'll build on that foundation to explore how to work with collections of data, process them efficiently with iterators, and handle errors gracefully using Rust's `Result` type and the `?` operator. These concepts are used extensively in real-world Rust code, including in our `rs-wsProxy` project where we see patterns like iterator chains for building allowlists, HashMaps for redirect maps, and `Result` types for network operations.

<!--more-->

## Vec<T>: Growable Arrays

The `Vec<T>` type is Rust's growable array type. It stores elements of type `T` contiguously on the heap, making it efficient for iteration and indexed access. Unlike arrays, vectors can grow and shrink in size at runtime.

### Creating Vectors

You can create an empty vector with `Vec::new()` or use the `vec!` macro for initialization:

```rust
let mut numbers: Vec<i32> = Vec::new();
// or
let numbers = vec![1, 2, 3, 4, 5];
```

The `mut` keyword is necessary if you plan to modify the vector after creation. The type annotation `: Vec<i32>` is often unnecessary due to type inference when using `vec!`.

### Adding and Removing Elements

Use `push` to add elements to the end and `pop` to remove and return the last element:

```rust
let mut numbers = vec![1, 2, 3];
numbers.push(4); // [1, 2, 3, 4]
let last = numbers.pop(); // Some(4), numbers is now [1, 2, 3]
```

To insert or remove at specific positions, use `insert` and `remove`:

```rust
numbers.insert(1, 99); // [1, 99, 2, 3]
let second = numbers.remove(1); // 99, numbers is now [1, 2, 3]
```

Note that `insert` and `remove` shift all subsequent elements, which can be slow for large vectors.

### Accessing Elements

You can access elements by index using square brackets, but this will panic if the index is out of bounds:

```rust
let third = numbers[2]; // Panics if numbers has fewer than 3 elements
```

For safe access, use the `get` method which returns an `Option<&T>`:

```rust
match numbers.get(2) {
    Some(value) => println!("The third element is {}", value),
    None => println!("There is no third element"),
}
```

### Iterating Over Vectors

You can iterate over a vector in three ways, depending on whether you want to take ownership, borrow immutably, or borrow mutably:

```rust
// Immutable borrow - iter()
for num in &numbers {
    println!("{}", num);
}

// Mutable borrow - iter_mut()
for num in &mut numbers {
    *num *= 2;
}

// Taking ownership - into_iter()
for num in numbers {
    println!("{}", num);
    // numbers is now empty and cannot be used again
}
```

### Ownership Considerations

When you iterate with `&numbers`, you borrow the vector immutably, so you can't modify it during iteration. With `&mut numbers`, you get mutable references to each element. With `numbers` (into_iter), you take ownership of the vector and consume it, moving each element out.

## HashMap<K, V>: Key-Value Stores

A `HashMap<K, V>` stores mappings from keys of type `K` to values of type `V`. It provides average O(1) lookup, insertion, and removal times.

### Creating and Inserting

Create a new hash map with `HashMap::new()` or collect from an iterator:

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);
```

### Accessing Values

The `get` method returns an `Option<&V>`:

```rust
match scores.get("Blue") {
    Some(score) => println!("Blue's score: {}", score),
    None => println!("No team named Blue"),
}
```

### Iterating

You can iterate over keys, values, or key-value pairs:

```rust
for (key, value) in &scores {
    println!("{}: {}", key, value);
}
```

### The Entry API

The `entry` API is useful for inserting a value only if a key doesn't already exist, or for updating a value based on its current value:

```rust
// Insert if key doesn't exist
scores.entry(String::from("Green")).or_insert(30);

// Update based on current value
scores.entry(String::from("Blue")).and_modify(|e| *e += 10);
```

### Ownership of Keys and Values

For types that implement `Copy` (like integers), values are copied into the hash map. For owned types like `String`, the hash map takes ownership of the value:

```rust
let field_name = String::from("Favorite color");
let field_value = String::from("Blue");

let mut map = HashMap::new();
map.insert(field_name, field_value);
// field_name and field_value are now invalid; they've been moved into the map
```

To retain ownership, insert references instead, but then the data you're pointing to must live at least as long as the hash map.

## String and &str: Owned vs Borrowed Strings

Rust has two main string types: `String` (owned, growable, heap-allocated) and `&str` (borrowed slice, view into existing string data).

### String: Owned String Data

A `String` is a growable, mutable, owned UTF-8 string:

```rust
let mut s = String::from("hello");
s.push_str(", world!");
s.push('!');
// s is now "hello, world!"
```

You can concatenate strings with `+` or the `format!` macro:

```rust
let s1 = String::from("Hello");
let s2 = String::from("world!");
let s3 = s1 + &s2; // s1 is moved, s2 is borrowed
let s4 = format!("{} {}", s1, s2); // s1 and s2 are unchanged
```

Note that `+` takes ownership of the left operand and borrows the right, which is why we often see `&s2`.

### &str: Borrowed String Slices

A string slice `&str` is a view into a string, typically a UTF-8 slice. String literals are `&'static str`:

```rust
let hello = "Hello, world!"; // &'static str
let s = String::from("hello world");
let hello = &s[0..5]; // "hello"
let world = &s[6..11]; // "world"
```

String slices must be valid UTF-8. You can create them from `String` or string literals.

### Why Two Types?

The distinction exists because of Rust's ownership system. `String` owns its data and can modify it, while `&str` is a view that doesn't own the data. Use `String` when you need to own or modify string data, and `&str` when you just need to read or view string data.

This distinction appears frequently in APIs: functions that need to own or modify strings take `String`, while functions that only need to read string data take `&str`.

## Iterators: Lazy, Composable Iteration

The `Iterator` trait is the foundation of Rust's iterator model. An iterator produces a sequence of values, and you can chain iterator adapters to transform sequences lazily.

### Creating Iterators

Vectors, slices, strings, and hash maps all provide iterator methods:

```rust
let v = vec![1, 2, 3];
// Immutable iteration
let iter = v.iter();
// Mutable iteration
let iter_mut = v.iter_mut();
// Into iteration (takes ownership)
let into_iter = v.into_iter();
```

### Iterator Adaptors

Iterator adaptors transform iterators into new iterators without consuming them until you call a consuming adaptor:

```rust
let v = vec![1, 2, 3, 4, 5];
// Map: transform each element
let doubled: Vec<i32> = v.iter().map(|x| x * 2).collect();
// Filter: keep only elements that match a predicate
let evens: Vec<i32> = v.iter().filter(|x| x % 2 == 0).collect();
// Chain: combine multiple adaptors
let result: Vec<i32> = v.iter()
    .filter(|x| x % 2 == 0)
    .map(|x| x * 3)
    .collect();
// result is [6, 12, 18]
```

### Consuming Adaptors

Consuming adaptors consume the iterator and produce a result:

- `collect()`: gathers items into a collection
- `fold()`: accumulates items into a single value
- `find()`: finds the first element matching a predicate
- `any()`/`all()`: test if any/all elements match a predicate

### Zero-Cost Abstractions

Iterator chains compile down to efficient loops equivalent to hand-written code. The compiler optimizes away the iterator overhead, making chains like `.filter().map().collect()` as fast as a manual loop.

## Error Handling with Result<T, E>

Rust doesn't have exceptions. Instead, it uses the `Result<T, E>` type for recoverable errors and `panic!` for unrecoverable errors.

### The Result Type

`Result<T, E>` is an enum with two variants:

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

### Pattern Matching

You can handle results with pattern matching:

```rust
fn parse_number(s: &str) -> Result<i32, std::num::ParseIntError> {
    s.parse::<i32>()
}

match parse_number("42") {
    Ok(n) => println!("Number: {}", n),
    Err(e) => println!("Failed to parse: {}", e),
}
```

### The ? Operator

The `?` operator simplifies error propagation. If the result is `Ok`, it returns the inner value; if it's `Err`, it returns the error from the function:

```rust
fn parse_and_double(s: &str) -> Result<i32, std::num::ParseIntError> {
    let n = s.parse::<i32>?; // Returns early if Err
    Ok(n * 2)
}
```

The `?` operator can be used in any function that returns `Result<T, E>` (or `Option<T>`).

### unwrap() and expect()

For quick prototyping, you can use `unwrap()` (panics on `Err`) or `expect()` (panics with a custom message):

```rust
let n = s.parse::<i32>().expect("Failed to parse number");
```

Use these sparingly in production code.

### Converting Between Error Types

When calling functions that return different error types, you may need to convert errors:

```rust
use std::fs::File;
use std::io;
use std::num;

fn read_number_from_file(path: &str) -> Result<i32, Box<dyn std::error::Error>> {
    let mut file = File::open(path)?; // io::Error
    let mut contents = String::new();
    file.read_to_string(&mut contents)?; // io::Error
    let n = contents.trim().parse::<i32>()?; // num::ParseIntError
    Ok(n)
}
```

The `?` operator automatically converts errors using `From::from` when the return type is a trait object like `Box<dyn std::error::Error>`.

### Custom Error Types

For larger projects, define your own error type:

```rust
use std::fmt;

#[derive(Debug)]
enum AppError {
    Io(io::Error),
    Parse(num::ParseIntError),
    NotFound,
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "IO error: {}", e),
            AppError::Parse(e) => write!(f, "Parse error: {}", e),
            AppError::NotFound => write!(f, "Item not found"),
        }
    }
}

impl std::error::Error for AppError {}

impl From<io::Error> for AppError {
    fn from(e: io::Error) -> Self {
        AppError::Io(e)
    }
}

impl From<num::ParseIntError> for AppError {
    fn from(e: num::ParseIntError) -> Self {
        AppError::Parse(e)
    }
}
```

Then use `?` to convert automatically:

```rust
fn process_data(path: &str) -> Result<(), AppError> {
    let data = std::fs::read_to_string(path)?; // Converts io::Error to AppError
    let value = data.trim().parse::<i32>()?;   // Converts ParseIntError to AppError
    println!("Value: {}", value);
    Ok(())
}
```

## Connection to rs-wsProxy

In our `rs-wsProxy` project, we see these concepts applied in real code:

### Iterator Chains in `build_allowed_list`

In `src/proxy.rs`, the `build_allowed_list` function uses an iterator chain to process a comma-separated list of origins:

```rust
fn build_allowed_list(s: &str) -> Vec<String> {
    s.split(',')
        .map(|s| s.trim())
        .filter(|s| !s.isEmpty())
        .map(|s| s.to_string())
        .collect()
}
```

This chain:
1. Splits the input string by commas
2. Trims whitespace from each piece
3. Filters out empty strings
4. Converts each piece to an owned `String`
5. Collects the results into a vector

### HashMap Construction in `build_redirects`

The `build_redirects` function creates a `HashMap` from a list of redirect rules:

```rust
fn build_redirects(rules: &[(&str, &str)]) -> HashMap<String, String> {
    let mut map = HashMap::new();
    for (from, to) in rules {
        map.insert(from.to_string(), to.to_string());
    }
    map
}
```

This could also be written using `collect` on an iterator:

```rust
fn build_redirects(rules: &[(&str, &str)]) -> HashMap<String, String> {
    rules.iter()
        .map(|(from, to)| (from.to_string(), to.to_string()))
        .collect()
}
```

### Error Handling with Result in `connect_tcp`

The `connect_tcp` function returns a `Result<TcpStream, String>` and uses the `?` operator for error propagation:

```rust
fn connect_tcp(addr: &str) -> Result<TcpStream, String> {
    TcpStream::connect(addr)
        .map_err(|e| format!("Failed to connect to {}: {}", addr, e))
}
```

Here, `TcpStream::connect` returns a `Result<TcpStream, io::Error>`, and we use `map_err` to convert the `io::Error` to a `String` to match our function's error type.

### Error Handling in `validate_tls_paths`

The `validate_tls_paths` function returns `Result<(), String>` and uses `?` to propagate errors:

```rust
fn validate_tls_paths(cert_path: &str, key_path: &str) -> Result<(), String> {
    // Check that both files exist
    std::fs::metadata(cert_path)?;
    std::fs::metadata(key_path)?;
    Ok(())
}
```

Each call to `std::fs::metadata` returns a `Result<Metadata, io::Error>`. The `?` operator converts the `io::Error` to a `String` (via `From::from` implementation we'd need to provide) and returns early if there's an error.

## Summary

We've covered Rust's core collection types (`Vec<T>`, `HashMap<K, V>`), string types (`String` vs `&str`), the powerful iterator system with its zero-cost abstractions, and Rust's approach to error handling with `Result<T, E>` and the `?` operator.

These concepts work together to enable safe, efficient, and expressive code. In `rs-wsProxy`, we see them applied in practical ways: processing configuration with iterators, building lookup tables with hash maps, and handling I/O and parsing errors with `Result`.

Continue your Rust journey with the next topic: [Traits & Generics](/posts/rust/rust-traits-generics/). There you'll learn how to write generic code that works with multiple types and how to define shared behavior with traits.

← Previous: [Structs, Enums & Pattern Matching](/posts/rust/rust-structs-enums/)