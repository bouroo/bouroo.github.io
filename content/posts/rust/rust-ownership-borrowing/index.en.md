---
title: "Ownership, Borrowing & Lifetimes"
subtitle: ""
date: 2026-07-08T09:00:00+07:00
lastmod: 2026-07-08T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "The heart of Rust: ownership, borrowing, references, and lifetimes that make Rust memory-safe without a garbage collector"
license: ""
images: []
tags: ["Rust", "Tutorial"]
categories: ["Rust"]
featuredImage: "featured-image.jpeg"
featuredImagePreview: "featured-image.jpeg"
lightgallery: true
---

Ownership is Rust's most distinctive feature, enabling memory safety guarantees without needing a garbage collector. This system enforces strict rules about how data is accessed and modified, preventing entire classes of bugs at compile time. Understanding ownership is essential for writing safe and efficient Rust code. This tutorial builds upon the concepts introduced in [Getting Started with Rust](/posts/rust/rust-getting-started/).

<!--more-->

## The Stack and the Heap

Memory in Rust is managed through two primary regions: the stack and the heap. The stack stores data with a fixed, known size at compile time and operates in a last-in, first-out manner, making it extremely fast. The heap stores data of unknown or changing size, requiring dynamic allocation and deallocation, which is slower but more flexible. Ownership rules govern how data moves between these regions, ensuring heap data is properly cleaned up when no longer needed.

Consider a string literal stored on the stack versus a `String` on the heap:

```rust
let s1 = "hello"; // &str: string literal, stored in binary, stack reference
let mut s2 = String::from("hello"); // String: heap allocation
s2.push_str(", world"); // Modifiable heap data
```

Here, `s1` is a string literal stored in the program's read-only memory, accessible via a stack-based reference. `s2` points to heap-allocated memory that can grow. When `s2` goes out of scope, its heap memory is automatically freed. This distinction is crucial: stack data is copied by value, while heap data requires explicit ownership transfers to prevent double-free errors.

## Ownership Rules

Rust's ownership system enforces three fundamental rules: each value has exactly one owner, there can only be one owner at a time, and when the owner goes out of scope, the value is dropped. These rules ensure memory safety without runtime overhead. The compiler enforces them at compile time, catching errors before runtime.

Consider this scope example demonstrating ownership and dropping:

```rust
{
    let s = String::from("hello"); // s enters scope
    // s is valid here
} // s goes out of scope, drop is called, memory freed
```

When `s` leaves the scope, Rust automatically calls `drop`, returning the heap memory to the system. If we tried to use `s` after the closing brace, we'd get a compile-time error. This automatic cleanup prevents memory leaks and dangling pointers, common issues in garbage-collected or manually managed languages.

## Move Semantics

When assigning a `String` to another variable, Rust moves ownership rather than copying the heap data. This prevents expensive deep copies and ensures only one owner exists. Attempting to use the original variable after a move results in a compile-time error.

Here's a classic move semantics example:

```rust
let s1 = String::from("hello");
let s2 = s1; // Ownership moves to s2

println!("{}", s1); // ERROR: borrow of moved value: `s1`
```

After `s2 = s1`, `s1` is no longer valid. Unlike types that implement the `Copy` trait (like integers), `String` owns heap data, so moving is more efficient than copying. To explicitly duplicate heap data, use the `clone` method:

```rust
let s1 = String::from("hello");
let s2 = s1.clone(); // Deep copy of heap data

println!("{}", s1); // OK: s1 still valid
println!("{}", s2); // OK: s2 has its own copy
```

Functions also follow move semantics when taking ownership of parameters:

```rust
fn takes_ownership(s: String) {
    println!("{}", s);
} // s is dropped here

let s = String::from("hello");
takes_ownership(s);
// println!("{}", s); // ERROR: value moved
```

To return ownership from a function, simply return the `String`:

```rust
fn gives_ownership() -> String {
    let s = String::from("hello");
    s // s is returned, moving ownership to caller
}

let s = gives_ownership(); // s owns the returned String
```

## References and Borrowing

Instead of taking ownership, functions can borrow references using `&` (immutable) or `&mut` (mutable). Immutable references allow multiple readers, while mutable references require exclusive access. The borrowing rules prevent data races at compile time: you can have either many immutable references or exactly one mutable reference to a particular piece of data in a particular scope.

Consider these borrowing examples:

```rust
let s = String::from("hello");

let len1 = calculate_length(&s); // Immutable borrow
let len2 = calculate_length(&s); // Another immutable borrow - OK

fn calculate_length(s: &String) -> usize {
    s.len()
} // s goes out of scope, but &s does not own the data
```

Attempting to create a mutable reference while an immutable one exists causes a compile error:

```rust
let mut s = String::from("hello");

let r1 = &s; // Immutable borrow
let r2 = &s; // Another immutable borrow - OK
let r3 = &mut s; // ERROR: cannot borrow as mutable while also borrowed as immutable

println!("{}, {}, {}", r1, r2, r3);
```

To fix this, ensure mutable borrows are exclusive:

```rust
let mut s = String::from("hello");

let r1 = &s; // OK
let r2 = &s; // OK
println!("{} {}", r1, r2); // r1 and r2 no longer used after this point

let r3 = &mut s; // OK: previous immutable borrows are not used here
println!("{}", r3);
```

## Slices

Slices let you reference a contiguous sequence of elements in a collection without taking ownership. A string slice (`&str`) references part of a `String`, while a slice `[T]` works for arrays or vectors. Slices are fat pointers containing a pointer to the data and a length, enabling safe, efficient access to portions of data.

String slices are particularly useful for working with string literals and substrings:

```rust
let s = String::from("hello world");

let hello = &s[0..5]; // Reference to bytes 0..5 (not including 5)
let world = &s[6..11]; // Reference to bytes 6..11

// String literals are slices
let literal = "hello world"; // &str type

// String slice of entire string
let whole = &s[..]; // Equivalent to &s[0..s.len()]
```

Attempting to create an invalid slice (out-of-bounds or invalid UTF-8 boundary) causes a panic:

```rust
let s = String::from("hello");

// let oops = &s[0..10]; // PANIC: index out of bounds
// For UTF-8, slicing in the middle of a character panics:
let hello = "Здравствуйте"; // Russian for "hello"
// let oops = &hello[0..1]; // PANIC: byte index 1 is not a char boundary
```

Array slices work similarly:

```rust
let a = [1, 2, 3, 4, 5];

let slice = &a[1..3]; // Reference to elements at index 1 and 2
assert_eq!(slice, &[2, 3]);

// Slices coerce to []T automatically in many contexts
fn print_slice(slice: &[i32]) {
    println!("{:?}", slice);
}

print_slice(&a); // Works because &[i32; 5] coerces to &[i32]
```

## Connection to rs-wsProxy

In the `rs-wsProxy` project, ownership and borrowing principles directly influence design. The `connect_tcp` function takes an address as a string slice (`&str`), borrowing the address without taking ownership, which is efficient for configuration strings that often live for the program's lifetime:

```rust
// Simplified from rs-wsProxy
fn connect_tcp(addr: &str) -> std::io::Result<TcpStream> {
    TcpStream::connect(addr)
}

// Usage:
let addr = "127.0.0.1:8080".to_string();
connect_tcp(&addr); // Borrow the String as &str
// addr remains valid after the call
```

For sharing state across asynchronous tasks, `Arc<AppState>` provides thread-safe shared ownership. `Arc` (Atomic Reference Counting) allows multiple threads to own a value, incrementing/decrementing an atomic counter to track references. When the count reaches zero, the value is dropped:

```rust
use std::sync::Arc;

struct AppState {
    config: Config,
    stats: Statistics,
}

// In main or initialization:
let state = Arc::new(AppState {
    config: load_config(),
    stats: Statistics::new(),
});

// When spawning tasks:
let state_clone = Arc::clone(&state); // Increments reference count
tokio::spawn(async move {
    // Use state_clone here
    // When this task ends, state_clone is dropped, decrementing the count
});

// Original state can still be used here
```

This pattern ensures `AppState` remains valid as long as at least one task holds a reference, preventing use-after-free in concurrent scenarios.

## Summary

Ownership, borrowing, and lifetimes form Rust's guarantee of memory safety without garbage collection. Ownership ensures each value has a single owner responsible for cleanup. Borrowing allows temporary access via references, governed by strict rules preventing data races. Slices provide efficient views into data without ownership transfer. Lifetimes ensure references remain valid for their intended use. Together, these concepts enable fearless concurrency and predictable resource management.

Understanding these concepts is crucial for effective Rust programming, as evidenced in real-world projects like `rs-wsProxy` where they enable safe, high-performance networking code. Mastering ownership transforms how you think about memory and data relationships, leading to more robust and efficient systems.

← Previous: [Getting Started with Rust](/posts/rust/rust-getting-started/)
Next: [Structs, Enums & Pattern Matching](/posts/rust/rust-structs-enums/)