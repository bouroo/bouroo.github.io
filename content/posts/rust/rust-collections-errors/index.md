---
title: "Collections, Iterators และ Error Handling"
subtitle: ""
date: 2026-07-10T09:00:00+07:00
lastmod: 2026-07-10T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "เรียนรู้ Vec, HashMap, Iterators และการจัดการ Error ด้วย Result และ ? operator"
license: ""
images: []
tags: ["Rust", "Tutorial"]
categories: ["Rust"]
featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"
lightgallery: true
---

การเขียนโปรแกรมใน Rust ไม่ได้หยุดอยู่แค่การเขียนฟังก์ชันและโครงสร้างข้อมูลพื้นฐานเท่านั้น แต่ยังต้องเข้าใจกลไกการจัดการข้อมูลที่ซับซ้อนขึ้น เช่น คอลเลกชันต่างๆ ที่ช่วยให้เราจัดการข้อมูลหลายๆ ชิ้นได้อย่างมีประสิทธิภาพ รวมถึงการใช้อิเทอร์เรเตอร์เพื่อประมวลผลข้อมูลเหล่านั้นอย่างมีประสิทธิภาพ และที่สำคัญที่สุดคือการจัดการข้อผิดพลาดอย่างเหมาะสมด้วยระบบ Result และตัวดำเนินการ ? ซึ่งเป็นหัวใจสำคัญของการเขียนโค้ดที่ปลอดภัยและน่าเชื่อถือใน Rust บทความนี้เป็นส่วนที่ 4 ของซีรีส์ Rust โดยต่อเนื่องจาก Part 3 ที่พูดถึง Structs, Enums และ Pattern Matching และจะนำไปสู่ Part 5 ที่จะพูดถึง Traits และ Generics ต่อไป เราจะเจาะลึกแต่ละหัวข้อด้วยคำอธิบายอย่างละเอียดและตัวอย่างโค้ดที่สามารถนำไปใช้ได้จริง

<!--more-->

## Vec<T>

`Vec<T>` คือเวกเตอร์แบบไดนามิกที่เก็บข้อมูลหลายชนิดไว้ด้วยกันบนฮีป เหมือนกับ `ArrayList` ใน Java หรือ `slice` ใน Go มันสามารถขยายขนาดได้โดยอัตโนมัติเมื่อต้องการเก็บข้อมูลเพิ่ม และเป็นคอลเลกชันที่ใช้บ่อยที่สุดใน Rust เนื่องจากความยืดหยุ่นและประสิทธิภาพ

```rust
// สร้างเวกเตอร์ว่างของจำนวนเต็ม i32
let mut numbers: Vec<i32> = Vec::new();
// เพิ่มค่าเข้าไปในเวกเตอร์
numbers.push(1);
numbers.push(2);
numbers.push(3);

// อ่านค่าดัชนีแรก - จะเกิด panic หากอยู่นอกขอบเขต
println!("First: {}", numbers[0]); // First: 1
// ตรวจสอบความยาวของเวกเตอร์
println!("Length: {}", numbers.len()); // Length: 3

// ใช้ macro vec! เพื่อสร้างเวกเตอร์พร้อมค่าเริ่มต้น
let zeros = vec![0; 5]; // สร้างเวกเตอร์ [0, 0, 0, 0, 0]
```

การอ่านค่าจากเวกเตอร์มีสองวิธีหลัก ได้แก่ การใช้ดัชนี `[index]` ซึ่งจะทำให้เกิด panic หากอยู่นอกขอบเขต และการใช้เมธอด `.get()` ซึ่งคืนค่าเป็น `Option<&T>` ทำให้สามารถจัดการกับกรณีที่อยู่นอกขอบเขตได้อย่างปลอดภัยโดยไม่ทำให้โปรแกรมหยุดทำงานอย่างกะทันหัน

```rust
let fruits = vec!["apple", "banana", "cherry"];

// วิธีที่ 1: การเข้าถึงด้วยดัชนี - จะ panic หากดัชนีอยู่นอกขอบเขต
let first = fruits[0]; // "apple"

// วิธีที่ 2: การใช้ get - ปลอดภัย คืนค่าเป็น Option
match fruits.get(10) {
    Some(fruit) => println!("Found: {}", fruit),
    None => println!("No fruit at index 10"),
}

// ยังสามารถใช้ if let เพื่อความกระชับได้อีกด้วย
if let Some(fruit) = fruits.get(1) {
    println!("Second fruit: {}", fruit);
} else {
    println!("No second fruit");
}
```

การลบค่าออกจากเวกเตอร์สามารถทำได้หลายวิธี เช่น `pop()` เพื่อลบและคืนค่าตัวสุดท้าย `remove(index)` เพื่อลบและคืนค่าตามดัชนีที่กำหนด หรือ `clear()` เพื่อลบทั้งหมด นอกจากนี้ยังสามารถลูปผ่านเวกเตอร์ได้โดยใช้อิเทอร์เรเตอร์ต่างๆ เช่น `.iter()` เพื่ออ่านค่าแบบอ้างอิง `.iter_mut()` เพื่อแก้ไขค่า หรือ `.into_iter()` เพื่อเป็นเจ้าของค่าและทำให้เวกเตอร์ว่างเปล่าหลังการลูป

```rust
let mut numbers = vec![1, 2, 3, 4, 5];

// ลบและคืนค่าตัวสุดท้าย
let last = numbers.pop(); // Some(5)
// ลบและคืนค่าตามดัชนีที่ 0
let first = numbers.remove(0); // 1
// ตอนนี้ numbers คือ [2, 3, 4]

// ลูปผ่านเวกเตอร์เพื่อพิมพ์ค่าทั้งหมด
for number in &numbers {
    println!("{}", number);
}

// ลูปเพื่อแก้ไขค่าในเวกเตอร์
for number in &mut numbers {
    *number *= 2; // คูณแต่ละค่าเป็น 2 เท่า
}
// ตอนนี้ numbers คือ [4, 6, 8]

// ลูปด้วย into_iter เพื่อเป็นเจ้าของค่า (ทำให้เวกเตอร์ว่างหลังลูป)
let sum: i32 = numbers.into_iter().sum();
// หลังจากนี้ numbers จะว่างเปล่าเพราะ into_iter ได้ย้ายความเป็นเจ้าของไป
```

## HashMap<K, V>

`HashMap<K, V>` คือตารางแฮชที่เก็บข้อมูลเป็นคู่ key-value โดยที่ key จะต้องเป็นแบบที่สามารถแฮชได้ (implements the `Hash` trait) และเทียบเท่าได้ (implements the `Eq` trait) มันให้ประสิทธิภาพในการค้นหา เฉลี่ย O(1) ทำให้เหมาะกับการเก็บข้อมูลที่ต้องการค้นหาโดยใช้ key บ่อยๆ คล้ายกับ Dictionary ใน Python หรือ Object ใน JavaScript แต่มีการรับประกันความปลอดภัยของหน่วยความจำตามแบบฉบับของ Rust

```rust
use std::collections::HashMap;

// สร้าง HashMap ว่างที่เก็บ String เป็น key และ i32 เป็น value
let mut scores: HashMap<String, i32> = HashMap::new();

// เพิ่มคู่ key-value เข้าไปใน HashMap
scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

// อ่านค่าจาก key - คืนค่าเป็น Option<&V>
let blue_score = scores.get("Blue");
// จะพิมพ์ว่า Blue's score: 10
match blue_score {
    Some(score) => println!("Blue's score: {}", score),
    None => println!("No score for Blue"),
}

// อัปเดตค่าของ key ที่มีอยู่แล้ว
scores.insert(String::from("Blue"), 25);
// ตอนนี้คะแนนของ Blue คือ 25

// ใช้ entry API เพื่อใส่ค่าเริ่มต้นหาก key ยังไม่มีอยู่
scores.entry(String::from("Green")).or_insert(30);
// หากไม่มี key "Green" จะใส่ค่า 30 เข้าไป
// หากมีอยู่แล้วจะไม่เปลี่ยนแปลงค่าเดิม
```

การทำงานกับคีย์และค่าใน HashMap มีเรื่องของการเป็นเจ้าของ (ownership) ที่สำคัญ เมื่อเราใส่ค่าแบบ `String` ลงใน HashMap HashMap จะเป็นเจ้าของค่านั้น หมายความว่าเราจะไม่สามารถใช้ตัวแปรเดิมที่เคยถือค่านั้นได้อีกต่อไปหลังจากที่ใส่เข้าไปแล้ว หากต้องการรักษาการเป็นเจ้าของไว้เราสามารถใช้อ้างอิง (`&String`) แทนได้ แต่ต้องแน่ใจว่าข้อมูลที่อ้างอิงนั้นมีอายุยืนยาวกว่า HashMap

```rust
use std::collections ฮ HashMap;

let field_name = String::from("Favorite color");
let field_value = String::from("Blue");

let mut map = HashMap::new();
// การใส่ String เข้าไปจะทำให้ HashMap เป็นเจ้าของ field_name และ field_value
map.insert(field_name, field_value);
// field_name และ field_value ไม่สามารถใช้งานได้อีกต่อไปที่นี่
// println!("{}", field_name); // จะเกิด error เพราะถูกย้ายไปแล้ว

// หากต้องการใช้ค่าเดิมต่อได้ ให้ใช้การอ้างอิงแทน
let field_name = String::from("Favorite color");
let field_value = String::from("Blue");

let mut map = HashMap::new();
// ใส่การอ้างอิงเข้าไป - แต่ต้องแน่ใจว่าข้อมูลที่อ้างอิงอยู่นานกว่า HashMap
map.insert(&field_name, &field_value);
// field_name และ field_value ยังสามารถใช้งานได้ต่อไป
println!("{}", field_name); // ปลอดภัย
```

การลูปผ่าน HashMap สามารถทำได้สามรูปแบบเหมือนกับเวกเตอร์ แต่จะได้คู่ key-value กลับมาเป็นทูเปิล เราสามารถเลือกลูปแบบอ้างอิงของทั้งคู่ อ้างอิงแบบเปลี่ยนแปลงได้ หรือเป็นเจ้าของทั้งคู่ ขึ้นอยู่กับความต้องการในการใช้งาน

```rust
use std::collections::HashMap;

let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

// ลูปแบบอ้างอิงเพื่ออ่านค่าเท่านั้น
for (key, value) in &scores {
    println!("{}: {}", key, value);
}

// ลูปแบบอ้างอิงที่เปลี่ยนแปลงได้เพื่อแก้ไขค่า
for (key, value) in &mut scores {
    if key == "Blue" {
        *value += 5; // เพิ่มคะแนน Blue ขึ้น 5
    }
}

// ลูปแบบเป็นเจ้าของ - จะทำให้ HashMap ว่างหลังจากลูปจบ
for (key, value) in scores {
    println!("{}: {}", key, value);
}
// ตอนนี้ scores ว่างเปล่าแล้ว
```

## String และ &str

ใน Rust มีสองประเภทหลักสำหรับการจัดการสตริง ได้แก่ `String` ซึ่งเป็นสตริงที่ถูกเป็นเจ้าของ เก็บไว้บนฮีป สามารถขยายขนาดได้ และสามารถแก้ไขได้ และ `&str` ซึ่งเป็นสตริงสไลซ์ที่เป็นการอ้างอิงไปยังส่วนหนึ่งของสตริงที่อยู่ที่ใดที่หนึ่ง ไม่ว่าจะเป็นในไบนารี่ (string literal) หรือใน `String` ที่ถูกเป็นเจ้าของอยู่ การเข้าใจความแตกต่างระหว่างสองประเภทนี้เป็นสิ่งสำคัญเพื่อหลีกเลี่ยงข้อผิดพลาดเกี่ยวกับการเป็นเจ้าของและการยืมข้อมูล

```rust
// สร้าง String ใหม่ที่ว่างเปล่า
let mut s = String::new();
// เพิ่มข้อความเข้าไปใน String
s.push_str("hello");
s.push('!'); // เพิ่มตัวอักษรเดียว
// ตอนนี้ s คือ "hello!"

// สร้าง String จาก literal โดยใช้ to_string() หรือ String::from
let hello = String::from("hello");
// หรือ
let hello = "hello".to_string();

// การต่อสตริงด้วย + หรือ format!
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = s1 + &s2; // s1 ถูกย้ายไปและไม่สามารถใช้งานได้อีกต่อไป
// หรือใช้ format! ซึ่งไม่ทำให้เกิดการย้าย
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = format!("{}{}", s1, s2); // s1 และ s2 ยังสามารถใช้งานได้ต่อไป

// การตัดสตริงเป็นสไลซ์ &str
let hello = String::from("hello world");
// สร้างสไลซ์จากดัชนี 0 ถึง 5 (ไม่รวม 5)
let hello = &hello[0..5]; // "hello"
// หรือจากดัชนี 6 ถึงสิ้นสุด
let world = &hello[6..]; // "world"
```

เหตุผลที่เราต้องมีทั้ง `String` และ `&str` ก็เพื่อให้ Rust สามารถจัดการหน่วยความจำได้อย่างปลอดภัยและมีประสิทธิภาพ `String` เป็นเจ้าของข้อมูลและรับผิดชอบในการปล่อยหน่วยความจำเมื่อหมดขอบเขต ในขณะที่ `&str` เป็นเพียงการอ้างอิงที่ไม่มีความรับผิดชอบในการปล่อยหน่วยความจำ ทำให้เราสามารถส่งต่อการอ้างอิงไปยังฟังก์ชันต่างๆ ได้โดยไม่ต้องกังวลเรื่องการย้ายหรือการคัดลอกข้อมูลที่มีขนาดใหญ่

```rust
// ฟังก์ชันที่รับ &str เป็นพารามิเตอร์สามารถรับได้ทั้ง String และ string literal
fn print_str(s: &str) {
    println!("{}", s);
}

// การใช้งาน
let owned = String::from("Hello");
let literal = "World";

print_str(&owned); // ต้องใช้ & เพื่อยืมการอ้างอิง
print_str(literal); // ส่งตรงได้เพราะ string literal เป็น &str อยู่แล้ว
```

## Iterators

อิเทอร์เรเตอร์ใน Rust เป็นวิธีที่ทรงพลังในการประมวลผลชุดข้อมูลแบบลำดับโดยไม่ต้องกังวลเกี่ยวกับการจัดการดัชนีหรือเงื่อนไขการหยุดลูป มันขึ้นอยู่กับทเรท `Iterator` ที่กำหนดเมธอด `next()` ซึ่งจะคืนค่า `Option<Item>` โดยเมื่อหมดองค์ประกอบจะคืนค่า `None` ความสวยงามของอิเทอร์เรเตอร์อยู่ที่การเป็น zero-cost abstraction หมายความว่าการใช้งานมันจะเร็วเท่ากับการเขียนลูปด้วยมือ แต่มีความปลอดภัยและความกระชับมากกว่า

```rust
let v1 = vec![1, 2, 3];

// สร้างอิเทอร์เรเตอร์แบบอ้างอิงเพื่ออ่านค่า
let v1_iter = v1.iter();

// ลูปผ่านอิเทอร์เรเตอร์ด้วย for loop (ซึ่งจะเรียก .into_iter() ภายใต้ฝากระโปรงสำหรับคอลเลกชัน)
for val in v1_iter {
    println!("Got: {}", val);
}

// หรือใช้เมธอดของอิเทอร์เรเตอร์โดยตรงผ่านการเรียกเมธอดซ้อนกัน (method chaining)
let v2: Vec<i32> = v1.iter()
    .map(|x| x + 1)     // เพิ่มค่าแต่ละตัวขึ้น 1
    .filter(|x| *x % 2 == 0) // กรองเฉพาะเลขคู่
    .collect();         // รวบรวมผลลัพธ์เป็นเวกเตอร์ใหม่
// v2 จะเป็น [2, 4]
```

อิเทอร์เรเตอร์มีเมธอดที่เรียกว่าอะแดปเตอร์ (adapters) ซึ่งสามารถนำมาเรียงกันเป็นลำดับเพื่อทำการแปลงข้อมูลที่ซับซ้อนได้โดยยังคงประสิทธิภาพสูง เมธอดเหล่านี้ได้แก่ `.map()` สำหรับการแปลงแต่ละองค์ประกอบ `.filter()` สำหรับการกรองเงื่อนไข `.take()` สำหรับการจำกัดจำนวนองค์ประกอบ และ `.collect()` สำหรับการรวบรวมผลลัพธ์ลงในคอลเลกชัน เช่น เวกเตอร์ หรือแฮชแมป

```rust
let numbers = vec![1, 2, 3, 4, 5, 6];

// หาผลรวมของเลขคู่ที่ยกกำลังสองที่น้อยกว่า 20
let sum_of_squares: i32 = numbers.iter()
    .filter(|&x| x % 2 == 0)   // เลือกเฉพาะเลขคู่
    .map(|x| x * x)            // ยกกำลังสอง
    .take_while(|&x| x < 20)   // หยุดเมื่อค่ามากกว่าหรือเท่ากับ 20
    .sum();                    // คำนวณผลรวม
// ขั้นตอน: [2,4,6] -> [4,16,36] -> [4,16] (เพราะ 36 >= 20) -> ผลรวม = 20
```

เมธอด `.into_iter()` จะทำให้เราเป็นเจ้าของค่าในคอลเลกชัน ซึ่งเหมาะสมเมื่อเราต้องการเปลี่ยนแปลงหรือใช้ค่าที่ได้จากการลูปโดยไม่ต้องกังวลเรื่องการยืมอ้างอิง ในขณะที่ `.iter_mut()` จะให้เราสามารถแก้ไขค่าในคอลเลกชันได้โดยตรงผ่านการอ้างอิงที่เปลี่ยนแปลงได้

```rust
let mut names = vec!["Alice".to_string(), "Bob".to_string(), "Charlie".to_string()];

// ใช้ iter_mut เพื่อเปลี่ยนชื่อทั้งหมดให้เป็นตัวพิมพ์ใหญ่
for name in names.iter_mut() {
    *name = name.to_uppercase();
}
// ตอนนี้ names คือ ["ALICE", "BOB", "CHARLIE"]

// ใช้ into_iter เพื่อเป็นเจ้าของชื่อและสร้างเวกเตอร์ใหม่ของความยาวชื่อ
let name_lengths: Vec<usize> = names.into_iter()
    .map(|name| name.len())
    .collect();
// หลังจากนี้ names จะไม่สามารถใช้งานได้อีกต่อไปเพราะถูกย้ายไปแล้ว
// name_lengths คือ [5, 3, 7]
```

## Error Handling ด้วย Result<T, E>

ใน Rust การจัดการข้อผิดพลาดทำผ่านประเภท `Result<T, E>` ซึ่งเป็นเอ็นัมที่มีสองแขน: `Ok(T)` สำหรับกรณีสำเร็จที่มีค่าผลลัพธ์ของประเภท T และ `Err(E)` สำหรับกรณีล้มเหลวที่มีข้อผิดพลาดของประเภท E วิธีนี้บังคับให้ผู้เขียนโค้ดต้องจัดการกับกรณีที่อาจล้มเหลวอย่างชัดเจน ป้องกันการละเลยข้อผิดพลาดโดยไม่ได้ตั้งใจ ซึ่งแตกต่างจากการใช้ข้อยกเว้นในภาษาอื่นๆ ที่อาจถูกมองข้ามได้

```rust
use std::fs::File;

// ฟังก์ชันที่อ่านไฟล์และคืนค่า Result<File, std::io::Error>
fn open_file(filename: &str) -> Result<File, std::io::Error> {
    let f = File::open(filename);
    f
}

// การใช้งานฟังก์ชันข้างต้นด้วยการจับคู่รูปแบบ (pattern matching)
match open_file("hello.txt") {
    Ok(file) => println!("File opened successfully"),
    Err(e) => println!("Failed to open file: {}", e),
}
```

เพื่อให้การเขียนโค้ดที่จัดการกับข้อผิดพลาดสะดวกุล่ิน Rust มีตัวดำเนินการ `?` ซึ่งสามารถใช้ได้เฉพาะในฟังก์ชันที่คืนค่าเป็น `Result` หรือ `Option` ตัวดำเนินการนี้จะทำหน้าที่เหมือนกับการจับคู่รูปแบบแต่ในรูปแบบที่กระชับกว่า: หากผลลัพธ์เป็น `Ok` จะคืนค่าที่อยู่ภายในออกมา แต่หากเป็น `Err` จะคืนค่าข้อผิดพลาดนั้นออกจากฟังก์ชันทันที (early return) ทำให้โค้ดอ่านและเขียนได้ง่ายขึ้นมาก

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();

    // เปิดไฟล์ หากล้มเหลวจะคืนค่าข้อผิดพลาดทันที
    let mut file = File::open("username.txt")?;
    // อ่านข้อมูลจากไฟล์ลงใน string หากล้มเหลวจะคืนค่าข้อผิดพลาดทันที
    file.read_to_string(&mut username)?;
    // หากทุกอย่างสำเร็จ จะคืนค่า Ok(username)
    Ok(username)
}
```

แม้ว่า `?` จะสะดวก แต่ก็ยังมีเมธอด `unwrap()` และ `expect()` ที่สามารถใช้ได้เมื่อเรามั่นใจว่าการดำเนินการนั้นจะไม่ล้มเหลว เช่น เมื่อเราแน่ใจว่าไฟล์มีอยู่จริงในช่วงเวลาทำงาน อย่างไรก็ตามการใช้เมธอดเหล่านี้ควรทำด้วยความระมัดระวังเพราะหากเกิดข้อผิดพลาดขึ้นจริง ๆ จะทำให้เกิด panic และหยุดการทำงานของโปรแกรมทันที `expect()` ช่วยให้เราสามารถระบุข้อความข้อผิดพลาดที่ต้องการแสดงได้เมื่อเกิด panic ซึ่งช่วยในการดีบักได้ดีกว่า `unwrap()` ที่ไม่มีข้อความอธิบาย

```rust
use std::fs::File;

// สมมติว่าเราแน่ใจว่าไฟล์นี้มีอยู่จริงในระบบไฟล์ของเรา
let f = File::open("essential_config.txt").unwrap();
// หรือพร้อมข้อความอธิบายเมื่อเกิดข้อผิดพลาด
let f = File::open("essential_config.txt")
    .expect("Failed to open essential_config.txt");
```

ในกรณีที่เราต้องการแปลงข้อผิดพลาดจากประเภทหนึ่งไปเป็นอีกประเภทหนึ่ง เช่น จาก `std::io::Error` เป็นข้อผิดพลาดเฉพาะของแอปพลิเคชันของเรา เราสามารถใช้เมธอด `.map_err()` บน `Result` เพื่อแปลงข้อผิดพลาดได้ นอกจากนี้ยังสามารถนิยามประเภทข้อผิดพลาดเฉพาะของตัวเองโดยใช้เอ็นัมและทำการแปลงโดยใช้ `From` trait เพื่อให้สามารถใช้ตัวดำเนินการ `?` ได้อย่างไร้รอยต่อ

```rust
use std::fmt;
use std::fs::File;
use std::io;

// นิยามประเภทข้อผิดพลาดเฉพาะของแอปพลิเคชัน
#[derive(Debug)]
enum AppError {
    Io(io::Error),
    NotFound,
}

// ทำให้ AppError สามารถแปลงจาก io.::Error ได้โดยอัตโนมัติ
impl From<io::Error> for AppError {
    fn from(err: io::Error) -> Self {
        AppError::Io(err)
    }
}

// ทำให้ App สามารถแสดงผลได้ด้วย fmt::Display
impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "I/O error: {}", e),
            AppError::NotFound => write!(f, "Resource not found"),
        }
    }
}

// ตอนนี้เราสามารถใช้ ? กับ AppError ได้โดยตรง
fn read_config_file() -> Result<String, AppError> {
    let mut content = String::new();
    File::open("config.txt")?.read_to_string(&mut content)?;
    Ok(content)
}
```

## เชื่อมโยงกับ rs-wsProxy

ในโครงการจริงอย่าง rs-wsProxy เราจะเห็นการประยุกต์ใช้แนวคิดทั้งหมดที่ได้พูดถึงไปแล้ว ตัวอย่างเช่น ในฟังก์ชัน `build_allowed_list` เราจะเห็นการใช้งานอิเทอร์เรเตอร์อย่างต่อเนื่องเพื่อแปลงและกรองข้อมูลจากสตริงที่คั่นด้วยเครื่องหมายจุลภาค รายการที่ว่างเปล่าจะถูกกรองออกและช่องว่างรอบๆ จะถูกตัดออก ก่อนจะเก็บผลลัพธ์ลงในเวกเตอร์ของสตริง

```rust
fn build_allowed_list(hosts: &str) -> Vec<String> {
    hosts.split(',')                    // แบ่งสตริงด้วยเครื่องหมายจุลภาค
        .map(|s| s.trim())              // ตัดช่องว่างที่อยู่ด้านหน้าและด้านหลัง
        .filter(|s| !s.is_empty())      // กรองออกสตริงที่ว่างเปล่า
        .map(|s| s.to_string())         // แปลงจาก &str เป็น String เพื่อเป็นเจ้าของข้อมูล
        .collect()                      // รวบรวมผลลัพธ์เป็น Vec<String>
}
```

ฟังก์ชัน `build_redirects` แสดงให้เห็นถึงการใช้งาน `HashMap` เพื่อสร้างความสัมพันธ์ระหว่างเส้นทางต้นทางและเส้นทางปลายทางในการตั้งค่าการเปลี่ยนเส้นทาง โดยใช้เมธอด `insert` เพื่อเพิ่มคู่ key-value ลงในแฮชแมป ซึ่งแสดงถึงการเป็นเจ้าของข้อมูลของทั้ง key และ value เมื่อพวกเขาเป็นประเภท `String`

```rust
use std::collections::HashMap;

fn build_redirects() -> HashMap<String, String> {
    let mut redirects = HashMap::new();
    redirects.insert(String::from("/old-path"), String::from("/new-target"));
    redirects.insert(String::from("/another/old"), String::from("/another/new"));
    redirects
}
```

ในส่วนของการจัดการข้อผิดพลาด ฟังก์ชัน `connect_tcp` แสดงให้เห็นถึงการใช้งาน `Result` และตัวดำเนินการ `?` เพื่อจัดการกับข้อผิดพลาดจากการเชื่อมต่อ TCP โดยฟังก์ชันนี้คืนค่าเป็น `Result<TcpStream, String>` ซึ่งหมายถึงหากสำเร็จจะได้ `TcpStream` หากล้มเหลวจะได้ข้อความข้อผิดพลาดเป็น `String` การใช้ `?` หลังจากการเรียก `TcpStream::connect` จะทำให้หากเกิดข้อผิดพลาดขึ้นฟังก์ชันจะคืนค่าข้อผิดพลาดนั้นทันทีโดยแปลงมันให้เป็น `String` ผ่านการใช้ `to_string()`

```rust
use std::net::TcpStream;
use std::io;

fn connect_tcp(addr: &str) -> Result<TcpStream, String> {
    // พยายามเชื่อมต่อ TCP หากล้มเหลวจะคืนค่า Err พร้อมข้อความข้อผิดพลาด
    TcpStream::connect(addr).map_err(|e| e.to_string())
}
```

สุดท้าย ฟังก์ชัน `validate_tls_paths` แสดงให้เห็นถึงการใช้งาน `Result<(), String>` เพื่อบ่งบอกว่าการตรวจสอบสำเร็จหรือล้มเหลว โดยไม่มีข้อมูลผลลัพธ์ใดๆ คืนกลับมาเมื่อสำเร็จ (ใช้หน่วยย่อย `()`) แต่เมื่อล้มเหลวจะคืนข้อความอธิบายข้อผิดพลาด ฟังก์ชันนี้ตรวจสอบว่ามีไฟล์ใบรับรองและคีย์ส่วนตัวอยู่หรือไม่ โดยใช้การจับคู่รูปแบบบนผลลัพธ์ของ `metadata()` เพื่อตรวจสอบว่าเป็นไฟล์จริงหรือไม่

```rust
use std::fs::metadata;
use std::io;

fn validate_tls_paths(cert_path: &str, key_path: &str) -> Result<(), String> {
    // ตรวจสอบไฟล์ใบรับรอง
    let cert_meta = metadata(cert_path).map_err(|e| format!("Failed to read cert: {}", e))?;
    if !cert_meta.is_file() {
        return Err("Cert path is not a file".to_string());
    }

    // ตรวจสอบไฟล์คีย์ส่วนตัว
    let key_meta = metadata(key_path).map_err(|e| format!("Failed to read key: {}", e))?;
    if !key_meta.is_file() {
        return Err("Key path is not a file".to_string());
    }

    Ok(())
}
```

## สรุป

ในบทความนี้เราได้สำรวจแนวคิดหลักๆ ของ Rust อย่างลึกซึ้ง ตั้งแต่การทำงานกับคอลเลกชันพื้นฐานอย่าง `Vec<T>` และ `HashMap<K, V>` ซึ่งแสดงให้เห็นถึงการจัดการหน่วยความจำและการเป็นเจ้าของที่ปลอดภัย ไปจนถึงการใช้อิเทอร์เรเตอร์เพื่อประมวลผลข้อมูลแบบลำดับอย่างมีประสิทธิภาพและเป็น zero-cost abstraction เรายังได้พูดถึงความแตกต่างที่สำคัญระหว่าง `String` และ `&str` ซึ่งเป็นหัวใจสำคัญของการทำงานกับข้อความใน Rust อย่างปลอดภัยและมีประสิทธิภาพ

นอกจากนี้เรายังได้เจาะลึกระบบการจัดการข้อผิดพลาดของ Rust ด้วยประเภท `Result<T, E>` และตัวดำเนินการ `?` ซึ่งบังคับให้เราต้องจัดการกับข้อผิดพลาดอย่างชัดเจน จึงช่วยลดโอกาสเกิดบั๊กจากการละเลยข้อผิดพลาด เราได้เห็นวิธีการแปลงข้อผิดพลาดและการสร้างประเภทข้อผิดพลาดเฉพาะเพื่อให้เหมาะกับบริบทของแอปพลิเคชัน

สุดท้ายเราได้เห็นการประยุกต์ใช้แนวคิดเหล่านี้ในโครงการจริงอย่าง rs-wsProxy ซึ่งแสดงให้เห็นว่าแนวคิดเหล่านี้ไม่ใช่เพียงทฤษฎีเท่านั้น แต่สามารถนำไปใช้แก้ปัญหาจริงๆ ได้อย่างมีประสิทธิภาพ ตั้งแต่การประมวลผลรายการโฮสต์ด้วยอิเทอร์เรเตอร์ การจัดการการเปลี่ยนเส้นทางด้วยแฮชแมป ไปจนถึงการจัดการข้อผิดพลาดในการเชื่อมต่อเครือข่ายและการตรวจสอบไฟล์ TLS

หากต้องการศึกษาต่อในหัวข้อต่อไปของซีรีส์นี้ โปรดดูบทความก่อนหน้าเกี่ยวกับ Structs, Enums และ Pattern Matching ที่ [Structs, Enums และ Pattern Matching](/posts/rust/rust-structs-enums/) และเตรียมตัวพบกับบทความถัดไปเกี่ยวกับ Traits และ Generics ที่ [Traits และ Generics](/posts/rust/rust-traits-generics/) ซึ่งจะพาเราไปสู่ความเข้าใจในระดับที่สูงขึ้นเกี่ยวกับระบบชนิดแบบทั่วไปและพอลิมอร์ฟิซึมใน Rust