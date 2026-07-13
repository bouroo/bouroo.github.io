---
title: "Traits & Generics"
subtitle: ""
date: 2026-07-11T09:00:00+07:00
lastmod: 2026-07-11T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Abstraction in Rust: traits, generics, trait bounds, and trait objects"
license: ""
images: []
tags: ["Rust", "Tutorial"]
categories: ["Rust"]
featuredImage: "featured-image.jpeg"
featuredImagePreview: "featured-image.jpeg"
lightgallery: true
---

Traits and generics are Rust's primary mechanisms for abstraction and code reuse. Traits define shared behavior, similar to interfaces in other languages, while generics allow code to work with multiple types. Together, they enable powerful abstractions without runtime cost. This tutorial builds on [Part 4: Collections, Iterators & Error Handling](/posts/rust/rust-collections-errors/) and prepares you for [Async Rust with Tokio](/posts/rust/rust-async-tokio/).

<!--more-->

## Defining Traits

Traits define shared behavior in an abstract way. Similar to interfaces in Java or interfaces in Go, Rust traits specify what methods a type must implement to possess certain behavior.

Let's define a simple `Summary` trait that provides a method to summarize content:

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

This trait declaration states that any type implementing `Summary` must provide a `summarize` method that takes an immutable reference to `self` and returns a `String`.

Now let's implement this trait for two structs: `NewsArticle` and `Tweet`.

```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

Here we've defined two structs and implemented the `Summary` trait for each. The `NewsArticle` implementation returns a formatted string with headline, author, and location, while `Tweet` returns username and content.

This approach is similar to interfaces in Go or Java, but with key differences:
- Rust traits can be implemented for types you don't own (the orphan rule has restrictions, but it's more flexible than Java's interfaces)
- Traits can have default method implementations (we'll see this next)
- Traits can be used as parameters through trait bounds (also coming up)

## Default Implementations

Traits can provide default implementations for their methods. Types implementing the trait can choose to use the default or override it.

Let's enhance our `Summary` trait with a default implementation:

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```

Now any type implementing `Summary` gets the default `summarize` method that returns "(Read more...)". However, we can still override it:

```rust
impl Summary for NewsArticle {
    // We're overriding the default implementation
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}

impl Summary for Tweet {
    // Using the default implementation
    // fn summarize(&self) -> String {
    //     String::from("(Read more...)")
    // }
}
```

In this example, `NewsArticle` overrides the default method while `Tweet` uses it. This provides flexibility - common behavior can be defined once in the trait, while specific types can customize when needed.

Default methods can also call other methods in the same trait, even if those methods don't have default implementations:

```rust
pub trait Summary {
    fn summarize_author(&self) -> String;

    fn summarize(&self) -> String {
        format!("(Read more from {}...)", self.summarize_author())
    }
}
```

Now any type implementing `Summary` only needs to implement `summarize_author` to get a default `summarize` method.

## Traits as Parameters

There are two ways to use traits as parameters in Rust: `impl Trait` syntax and trait bound syntax.

First, the `impl Trait` syntax - useful for simple cases:

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

This function accepts any type that implements the `Summary` trait. It's concise and readable for simple cases.

For more complex situations, we use trait bound syntax with generics:

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

Here we've introduced a generic type parameter `T` that is constrained by the `Summary` trait. This means `T` can be any type that implements `Summary`.

Which should you use? Use `impl Trait` for simple cases where you only need one parameter. Use trait bounds when:
- You need to use the same trait bound multiple times in a function signature
- You need to specify multiple trait bounds
- You want to be explicit about the generic nature of your function

Both approaches compile to the same code due to monomorphization (which we'll discuss later).

## Generics

Generics allow us to write code that works with multiple types. Let's start with a generic function that finds the largest element in a slice:

```rust
fn largest<T>(list: &[T]) -> T {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    *largest
}
```

This won't compile because we're using the `>` operator on type `T`, and not all types can be compared. We need to add a trait bound:

```rust
use std::cmp::PartialOrd;

fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    *largest
}
```

Now `T` must implement `PartialOrd` (for comparison) and `Copy` (so we can return a copy of the value). We could also return a reference (`&T`) to avoid the `Copy` requirement.

Let's look at generic structs:

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };
}
```

Here `Point<T>` is a generic struct that can hold any type for its coordinates, as long as both fields are the same type. We can also constrain the implementation:

```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
```

This implementation only exists for `Point<f32>`, not for other types.

Generic enums work similarly:

```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

These are fundamental to Rust's standard library.

### Monomorphization

When Rust compiles generic code, it performs monomorphization - generating specialized versions of the code for each concrete type used. For example, if we call `largest` with both `i32` and `f64`, the compiler generates two specialized functions:

```rust
fn largest_i32(list: &[i32]) -> i32 { /* ... */ }
fn largest_f64(list: &[f64]) -> f64 { /* ... */ }
```

This means generics have zero runtime cost - they're as fast as if you'd written the type-specific versions yourself. However, it does increase compile time and binary size.

## Trait Bounds

Sometimes we need multiple trait bounds. We can specify them with `+` syntax:

```rust
use std::fmt::Display;

fn notify(item: &(impl Summary + Display)) {
    println!("Breaking news! {}", item.summarize());
}
```

Or with trait bound syntax:

```rust
fn notify<T: Summary + Display>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

For longer lists of trait bounds, the `where` clause improves readability:

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone,
    U: Clone + Debug,
{
    // function body
}
```

The `where` clause is particularly useful when trait bounds become complex or when we want to keep the function signature clean.

## Trait Objects

So far we've discussed static dispatch - where the compiler knows exactly which method implementation to call at compile time. Rust also supports dynamic dispatch through trait objects.

A trait object is a pointer to an implementation of a trait, allowing for polymorphic behavior. We create trait objects using `dyn Trait`:

```rust
fn notify(item: &Box<dyn Summary>) {
    println!("Breaking news! {}", item.summarize());
}
```

Or more commonly with references:

```rust
fn notify(item: &dyn Summary) {
    println!("Breaking news! {}", item.summarize());
}
```

Here's how we might use it:

```rust
fn main() {
    let article = NewsArticle {
        headline: String::from("Penguins win the Stanley Cup Championship!"),
        location: String::from("Pittsburgh, PA, USA"),
        author: String::from("Iceburgh"),
        content: String::from(
            "The Pittsburgh Penguins once again are the best \
             hockey team in the NHL.",
        ),
    };

    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    let items: Vec<Box<dyn Summary>> = vec![
        Box::new(article),
        Box::new(tweet),
    ];

    for item in items {
        notify(&item);
    }
}
```

Notice we had to box our items to create trait objects. This is because trait objects have unknown size at compile time - we need indirection via a pointer.

### Static vs Dynamic Dispatch

Static dispatch (via generics and trait bounds) has zero runtime overhead but increases compile time and binary size. Dynamic dispatch (via trait objects) has a small runtime cost due to virtual method dispatch but keeps compile times lower and binary size smaller.

Choose static dispatch when:
- You know all possible types at compile time
- Performance is critical
- You want zero-cost abstractions

Choose dynamic dispatch when:
- You need heterogeneous collections (like our `Vec<Box<dyn Summary>>`)
- The exact types aren't known until runtime
- You prefer faster compilation over minimal binary size

## Standard Library Traits

Rust's standard library provides many useful traits. Let's examine some of the most important ones:

### Display and Debug

`Display` is for user-facing output, while `Debug` is for debugging:

```rust
use std::fmt;

struct Point {
    x: i32,
    y: i32,
}

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}

impl fmt::Debug for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Point {{ x: {}, y: {} }}", self.x, self.y)
    }
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{}", p);     // Output: (1, 2)
    println!("{:?}", p);
}
```

### Clone and Copy

`Clone` enables explicit duplication, while `Copy`T: Clone + Display>`.

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

impl Clone for Point {
    fn clone(&self) -> Self {
        Self {
            x: self.x,
            y: self.y,
        }
    }
}

// Or simply derive it:
#[derive(Debug, Clone)]
struct Point {
    x: i32,
    y: i32,
}
```

`Copy` is a marker trait for types that can be copied bit-for-wise. All `Copy` types must also be `Clone`:

```rust
#[derive(Debug, Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}
```

### From and Into

These traits provide a consistent way to convert between types:

```rust
use std::convert::From;

#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

impl From<(i32, i32)> for Point {
    fn from(point: (i32, i32)) -> Self {
        Point {
            x: point.0,
            y: point.1,
        }
    }
}

fn main() {
    let point = Point::from((3, 4));
    println!("{:?}", point); // Point { x: 3, y: 4 }
}
```

Thanks to the blanket implementation in the standard library, if `From<T> for U` is implemented, then `Into<U> for T` is automatically available:

```rust
fn main() {
    let point: Point = (3, 4).into();
    println!("{:?}", point); // Point { x: 3, y: 4 }
}
```

### Default

The `Default` trait provides a default value for a type:

```rust
#[derive(Debug, Default)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point::default();
    println!("{:?}", p); // Point { x: 0, y: 0 }
}
```

We can also implement it manually:

```rust
impl Default for Point {
    fn default() -> Self {
        Point { x: 0, y: 0 }
    }
}
```

### Derive Macros

Many standard library traits can be automatically implemented using the `derive` attribute. This reduces boilerplate and ensures correctness. Common derivable traits include:
- `Debug`
- `Clone`
- `Copy`
- `PartialEq`
- `Eq`
- `PartialOrd`
- `Ord`
- `Hash`
- `Default`

Custom derive macros are also possible through procedural macros, which we'll see in action with frameworks like `clap`.

## Connection to rs-wsProxy

In our rs-wsProxy project (built with Axum), traits are used extensively throughout the framework:

### Axum Traits
- `IntoResponse`: Converts types into HTTP responses. Our handlers return types that implement this trait.
- `FromRequest`: Extracts information from incoming requests (like headers, query parameters, or state).
- `FromRef`: Allows extracting shared state from application state.

Example from our project:
```rust
async fn handler(
    TypedHeader(headers): TypedHeader<headers::Authorization>,
    State(app_state): State<AppState>,
) -> impl IntoResponse {
    // ... handler logic
    Json(json!({ "status": "ok" }))
}
```

Here `TypedHeader<headers::Authorization>` uses `FromRequest` to extract the Authorization header, `State<AppState>` extracts application state, and `Json<T>` implements `IntoResponse` to produce a JSON response.

### clap Derive
We use `clap`'s derive feature to generate command-line argument parsing:
```rust
#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Args {
    #[arg(short, long)]
    port: u16,

    #[arg(short, long)]
    host: String,
}
```

This automatically implements the `Parser` trait for our `Args` struct, generating code to parse command-line arguments into this structure.

### Future Trait
Asynchronous programming in Rust is built around the `Future` trait. When we write `async fn`, the compiler transforms it into a struct that implements `Future`:

```rust
async fn handle_connection(mut stream: TcpStream) -> Result<()> {
    // ... async logic
}

// The compiler roughly transforms this to:
struct HandleConnectionFuture { /* fields */ }
impl Future for HandleConnectionFuture {
    type Output = Result<()>;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // ... polling logic
    }
}
```

This allows async functions to be composed and executed efficiently without runtime overhead.

## Summary

Traits and generics are fundamental to Rust's approach to abstraction:
- Traits define shared behavior (like interfaces in other languages)
- Generics enable type-agnostic code
- Trait bounds constrain generic types to those implementing specific traits
- Trait objects enable dynamic dispatch when needed
- The standard library provides essential traits like `Display`, `Debug`, `Clone`, `Copy`, `From`/`Into`, and `Default`
- Derive macros reduce boilerplate for common traits
- Real-world frameworks like Axum and clap leverage these concepts extensively

Understanding these concepts is crucial for writing idiomatic, efficient Rust code. They enable the zero-cost abstractions that make Rust powerful while maintaining safety and performance.

Next, we'll explore asynchronous programming with Tokio in [Async Rust with Tokio](/posts/rust/rust-async-tokio/). Previously, we covered [Collections, Iterators & Error Handling](/posts/rust/rust-collections-errors/).

Continue your Rust journey by exploring how these abstractions enable powerful asynchronous patterns in the next tutorial.