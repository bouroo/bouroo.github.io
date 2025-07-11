---
title: "Why Go vs Rust When You Can Have Go + Rust?"
subtitle: ""
date: 2023-10-13T15:38:13+07:00
lastmod: 2023-10-13T15:38:13+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Go ❤️ Rust"
license: ""
images: []

tags: ["Go", "Rust"]
categories: ["Go", "Rust"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

lightgallery: true
---

We often encounter the question of whether to use Go or Rust. After trying both Go and Rust for a while, I found that we can use cgo + Rust FFI. Since both have different advantages, we can just use both!

<!--more-->

# What is Rust FFI?
Rust FFI (Foreign Function Interface) is a tool that allows us to call functions and code in Rust from other languages such as C, C++, and Go smoothly and safely.

# Let's try it out
- Prerequisites
  - Install Rust from the [official website](https://www.rust-lang.org/tools/install)
  - Install Go from the [official website](https://go.dev/dl/)

## Create a Go Project
```md
go-rust-ffi
├── lib                 <--- Rust lib
│   └── rs-add
│       ├── src
│       │   └── lib.rs  <--- Rust lib src
│       ├── build.rs    <--- Rust build script
│       ├── Cargo.lock
│       └── Cargo.toml
├── go.mod
├── LICENSE
├── main.go             <--- Go file
├── Makefile            <--- Make script
└── README.md
```

In `lib`, we can use `cargo` to create a Rust project for us.
```bash
cd lib
cargo new --lib rs-add
```

We'll start with something simple like this in the `src/lib.rs` file:
```rust
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

Then we use `cbindgen` to generate a header file for us (or you can write it by hand).
```bash
cargo install cbindgen
cbindgen --lang c --output rs-add.h
```

Let's build the Rust lib.
```bash
cargo build --release
```
We will get `librs_add.so` and `librs_add.a`. For now, let's move them to `./lib` to avoid confusion.

In `main.go`, we can call it via cgo like this:
```go
package main

/*
#cgo LDFLAGS: -L./lib -l:librs_add.so
#include "./lib/rs-add.h"
*/
import "C"
import "fmt"

func main() {
    a := 10
    b := 20
    result := int(C.add(C.int(a), C.int(b)))
    fmt.Printf("Result: %d\n", result)
}
```

In the Go code, we use the `import "C"` statement to call the Rust function via FFI. This will use the header file to specify the `add` function and link the Rust lib to our Go code.

When building Go, we also need to include the lib.
```bash
CURRENT_DIR=$(pwd || echo ${PWD})
go build -ldflags="-r $(CURRENT_DIR)lib" -o dist/ ./...
```

## Conclusion
Now that we know about Rust FFI and how we can use Rust in our Go code via FFI + cgo to make Go and Rust communicate, we don't have to argue about whether to use Go or Rust anymore. We can take advantage of both, for example, using Go for the Router/Thread controller and Rust to help with hot functions. I have created an example that generates a QR code with Rust at [go-rust-ffi](https://github.com/bouroo/go-rust-ffi). You can go and play with it.

```