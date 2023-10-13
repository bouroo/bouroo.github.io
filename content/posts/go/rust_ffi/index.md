---
title: "จะ Go vs Rust ทำไมในเมื่อ Go + Rust ได้"
subtitle: ""
date: 2023-10-13T15:38:13+07:00
lastmod: 2023-10-13T15:38:13+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin-vir.pages.dev"
description: ""
license: ""
images: []

tags: ["Go", "Rust"]
categories: ["Go", "Rust"]

featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

lightgallery: true
---

เราจะเจอคำถามประเภทที่ว่าจะใช้อะไรดีระหว่าง Go กับ Rust หลังจากที่ได้ลองเขียนทั้ง Go กับ Rust มาสักพักแล้วพบว่าเราสามารถใช้ cgo + Rust FFI ได้ ซึ่งในเมื่อทั้งสองมีข้อดีต่างกัน เราก็ใช้มันทั้งสองไปเล๊ยยย

<!--more-->

# Rust FFI คืออะไร?
Rust FFI (Foreign Function Interface) คือตัวช่วยให้เราเรียกใช้ฟังก์ชันและโค้ดในภาษา Rust จากภาษาอื่น ๆ เช่น C, C++, และ Go ได้อย่างราบรื่นและปลอดภัย

# มาลองดูกัน
- สิ่งที่ต้องเตรียม
  - ติดตั้ง Rust จาก [เว็บไซต์หลัก](https://www.rust-lang.org/tools/install)
  - ติดตั้ง Go จาก [เว็บไซต์หลัก](https://go.dev/dl/)

## สร้าง Go Project
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

ซึ่งใน `lib` ใช้ `cargo` สร้าง Rust project ให้เราได้
```bash
cd lib
cargo new --lib rs-add
```

เราก็จะลองเริ่มจากอะไรที่ง่าย ๆ แบบนี้ที่ไฟล์ `src/lib.rs`
```rust
#[no_mangle]
pub extern "C" fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

แล้วเราก็ใช้ `cbindgen` สร้างไฟล์ header ให้เรา (หรือเขียนมือก็ได้)
```bash
cargo install cbindgen
cbindgen --lang c --output rs-add.h
```

build Rust lib กันเลย
```bash
cargo build --release
```
เราก็จะได้ `librs_add.so` กับ `librs_add.a` ในตอนนี้ให้ย้ายมาไว้ที่ `./lib` เพื่อความไม่งง

ส่วนใน `main.go` เราก็เรียกใช้ผ่าน cgo แบบนี้ได้เลย
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

ในโค้ด Go เราใช้คำสั่ง import "C" เพื่อเรียกใช้ฟังก์ชัน Rust ผ่าน FFI. ซึ่งจะใช้ไฟล์ header เพื่อระบุฟังก์ชัน add และ lib Rust เข้ากับ Go ของเรา

เวลา build Go เราก็เพิ่ม lib เข้าไปด้วย
```bash
CURRENT_DIR=$(pwd || echo ${PWD})
go build -ldflags="-r $(CURRENT_DIR)lib" -o dist/ ./...
```

## สรุป
เมื่อเราได้รู้จักกับ Rust FFI และวิธีที่เราสามารถใช้ Rust ใน Go ของเราผ่าน FFI + cgo เพื่อให้ Go และ Rust สื่อสารกันได้ เราก็ไม่ต้องเถียงกันแล้วว่า จะใช้อะไรดีระหว่าง Go กับ Rust ในเมื่อเราก็ใช้ประโยชน์จากทั้งสองได้ เช่น ใช้ Go ทำ Router/Thread controller แล้วเอา Rust มาช่วยทำในส่วน hot functions ซึ่งผมได้ลองทำตัวอย่างที่สร้าง QR ด้วย Rust ไว้ที่ [go-rust-ffi](https://github.com/bouroo/go-rust-ffi) แล้วไปยำกันได้