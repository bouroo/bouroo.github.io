---
title: "Traits และ Generics"
subtitle: ""
date: 2026-07-11T09:00:00+07:00
lastmod: 2026-07-11T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "abstraction ใน Rust: Traits, Generics, Trait bounds และ Trait objects"
license: ""
images: []
tags: ["Rust", "Tutorial"]
categories: ["Rust"]
featuredImage: "featured-image.jpeg"
featuredImagePreview: "featured-image.jpeg"
lightgallery: true
---

Traits และ Generics คือสองแนวคิดสำคัญที่ทำให้ Rust มีพลังในการเขียนโค้ดที่เป็น generic และสามารถแชร์พฤติกรรมระหว่างประเภทต่าง ๆ ได้อย่างปลอดภัยและมีประสิทธิภาพ บทความนี้เป็นภาคต่อจากภาค 4 ที่เราได้พูดถึง Collections, Iterators และ Error Handling มาแล้ว เราจะเจาะลึกว่า trait ทำหน้าที่เหมือน interface ในภาษาอื่น ๆ เช่น Go หรือ Java อย่างไร และ generics ช่วยให้เราเขียนฟังก์ชันและโครงสร้างข้อมูลที่ทำงานกับหลายประเภทได้โดยไม่สูญเสียประสิทธิภาพ เราจะพูดถึง trait bounds, trait objects รวมถึง trait ที่สำคัญในไลบรารีมาตรฐาน เช่น Display, Debug, Clone, Copy, From, Into และ Default แล้วเชื่อมโยงกับการใช้งานจริงในโปรเจกต์ rs-wsProxy ที่ใช้ Axum, clap และ async/await ด้วย

<!--more-->

## Traits คืออะไร

ใน Rust trait คือการกำหนดชุดของเมธอดที่ประเภทหนึ่งสามารถนำไปใช้งานได้ คล้ายกับ interface ในภาษาอื่น ๆ แต่ trait มีความยืดหยุ่นมากกว่าเพราะสามารถมี default implementation ได้ และสามารถนำไปใช้กับประเภทใดก็ได้ที่เราต้องการ ไม่ว่าจะเป็น struct ที่เราสร้างเองหรือประเภทจากไลบรารีมาตรฐาน เมื่อเราต้องการให้ประเภทหนึ่งมีพฤติกรรมบางอย่าง เราจะทำการ implement trait สำหรับประเภทนั้น ๆ ตัวอย่างเช่นเราอาจต้องการให้โครงสร้างข่าวสารและทวีตสามารถสรุปเนื้อหาได้ เราจึงกำหนด trait ชื่อ `Summary` ที่มีเมธอด `summarize` จากนั้นเราจะ implement trait นี้ให้กับ struct `NewsArticle` และ `Tweet` แต่ละตัวจะให้ผลลัพธ์ที่แตกต่างกันไปตามลักษณะของข้อมูล

ตัวอย่างการกำหนด trait และการ implement ดังต่อไปนี้

```rust
// ประกาศ trait Summary ที่มีเมธอด summarize คืนค่าเป็น String
pub trait Summary {
    fn summarize(&self) -> String;
}

// โครงสร้างข่าวสาร
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

// นำ Summary มาใช้กับ NewsArticle
impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, โดย {} ({})", self.headline, self.author, self.location)
    }
}

// โครงสร้างทวีต
pub struct Tweet {
    pub username: String,
    pub content: String,
    pub reply: bool,
    pub retweet: bool,
}

// นำ Summary มาใช้กับ Tweet
impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("{}: {}", self.username, self.content)
    }
}
```

เมื่อเรา implement trait ให้กับประเภทใด ๆ แล้ว เราสามารถเรียกใช้เมธอด `summarize` กับอินสแตนซ์ของประเภทนั้นได้เหมือนกับการเรียกเมธอดปกติ ตัวอย่างการใช้งาน:

```rust
fn main() {
    let article = NewsArticle {
        headline: String::from("Penguins win the Stanley Cup Championship!"),
        location: String::from("Pittsburgh, PA, USA"),
        author: String::from("Iceburgh"),
        content: String::from("The Pittsburgh Penguins once again are the best \
                                hockey team in the NHL."),
    };

    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    println!("บทความใหม่! {}", article.summarize());
    println!("ทวีตใหม่: {}", tweet.summarize());
}
```

การใช้ trait ช่วยให้เราสามารถเขียนฟังก์ชันที่ทำงานกับประเภทต่าง ๆ ได้ตราบใดที่ประเภทนั้น ๆ นำ trait ที่ต้องการมาใช้งาน นี่คือพื้นฐานของการเขียนโค้ดแบบ generic ใน Rust ที่เราจะพูดถึงในส่วนต่อไป

## Default implementations

Trait ไม่จำเป็นต้องประกาศเฉพาะเมธอดที่ไม่มีการ implement เท่านั้น เรายังสามารถให้ default implementation สำหรับเมธอดใน trait ได้อีกด้วย ซึ่งหมายความว่าประเภทใด ๆ ที่นำ trait มาใช้จะได้รับการ implement เมธอดนั้นโดยอัตโนมัติ หากประเภทนั้นไม่ได้ให้การ implement ของตัวเอง คุณสมบัตินี้ช่วยลดการเขียนโค้ดซ้ำซ้อนเมื่อหลายประเภทต้องการพฤติกรรมเดียวกัน แต่ยังอนุญาตให้แต่ละประเภท override ได้ถ้าต้องการพฤติกรรมที่แตกต่างออกไป

ตัวอย่างการเพิ่ม default implementation ใน trait `Summary`:

```rust
pub trait Summary {
    // Default implementation ที่คืนค่า String ว่างเปล่า
    fn summarize(&self) -> String {
        String::from("(อ่านเพิ่มเติม...)")
    }
}
```

เมื่อเรา implement trait สำหรับ `NewsArticle` เช่นเดิม แต่เราไม่ได้ override เมธอด `summarize` เมธอดจาก default จะถูกใช้แทน อย่างไรก็ตามหากเราต้องการให้ผลลัพธ์แตกต่างออกไปเราสามารถ override ได้โดยการให้ implementation ของตัวเอง ดังตัวอย่างต่อไปนี้

```rust
impl Summary for NewsArticle {
    // Override default implementation
    fn summarize(&self) -> String {
        format!("{}, โดย {} ({})", self.headline, self.author, self.location)
    }
}

// Tweet ไม่ได้ override จะใช้ default implementation
impl Summary for Tweet {}
```

เมื่อเราเรียกใช้ `summarize` กับอินสแตนซ์ของ `NewsArticle` จะได้ผลลัพธ์ตามที่เรา override แต่เมื่อเรียกกับ `Tweet` จะได้ default implementation ที่คืนค่า `(อ่านเพิ่มเติม...)` ซึ่งอาจไม่เหมาะสมสำหรับทวีต ดังนั้นในทางปฏิบัติเรามักจะให้แต่ละประเภท override default implementation เพื่อให้เหมาะสมกับข้อมูลของมัน

Default implementation ยังสามารถเรียกใช้เมธอดอื่นใน trait เดียวกันได้อีกด้วย ทำให้เราสามารถสร้างพฤติกรรมที่ซับซ้อนจากเมธอดพื้นฐานไม่กี่ตัวได้

## Traits เป็น parameter

เมื่อเราต้องการเขียนฟังก์ชันที่สามารถรับพารามิเตอร์ที่นำ trait บางอย่างมาใช้ได้ เรามีสองวิธีหลักในการระบุ trait bound ใน Rust วิธีแรกคือการใช้ `impl Trait` ในตำแหน่งพารามิเตอร์ ซึ่งเหมาะกับกรณีที่เราต้องการพารามิเตอร์เพียงตัวเดียวและไม่ต้องการระบุประเภทอย่างชัดเจน วิธีที่สองคือการใช้ trait bound แบบปกติด้วยรูปแบบ `T: Trait` ภายในวงเล็บเหลี่ยมของ generics ซึ่งให้ความยืดหยุ่นมากกว่าเมื่อต้องการระบุพารามิเตอร์หลายตัวหรือต้องการกำหนดเงื่อนไขเพิ่มเติม

ตัวอย่างการใช้ `impl Trait` เป็นพารามิเตอร์:

```rust
use crate::Summary;

// ฟังก์ชัน notify รับพารามิเตอร์ที่นำ Summary มาใช้
// โดยใช้รูปแบบ impl Trait
pub fn notify(item: &impl Summary) {
    println!("ข่าวด่วน! {}", item.summarize());
}
```

ฟังก์ชันนี้สามารถรับอินสแตนซ์ของ `NewsArticle` หรือ `Tweet` หรือประเภทอื่นใดก็ตามที่นำ `Summary` มาใช้ได้ โดยไม่ต้องระบุประเภทอย่างชัดเจน

อีกรูปแบบหนึ่งคือการใช้ trait bound แบบปกติ:

```rust
use crate::Summary;

// ฟังก์ชัน notify แบบใช้ trait bound ปกติ
pub fn notify<T: Summary>(item: &T) {
    println!("ข่าวด่วน! {}", item.summarize());
}
```

ที่นี่เราได้กำหนดว่า `T` ต้องนำ `Summary` มาใช้ ซึ่งทำให้เราสามารถระบุเงื่อนไขเพิ่มเติมได้ เช่น `T: Summary + Clone` เพื่อให้ `T` ต้องนำทั้ง `Summary` และ `Clone` มาใช้ หรือใช้ `where` clause เพื่อเพิ่มความอ่านง่ายเมื่อมีเงื่อนไขหลายอย่าง

การใช้ `impl Trait` เหมาะกับกรณีที่ฟังก์ชันมีพารามิเตอร์เพียงหนึ่งหรือสองตัวและไม่ต้องการเงื่อนไขซับซ้อน ในขณะที่ trait bound แบบปกติให้ความยืดหยุ่นมากกว่าเมื่อต้องการระบุหลายเงื่อนไขหรือต้องการใช้กับหลายพารามิเตอร์

## Generics

Generics คือกลไกที่ช่วยให้เราเขียนฟังก์ชัน โครงสร้างข้อมูล และเอ็นัมที่ทำงานกับหลายประเภทได้โดยไม่ต้องทำซ้ำโค้ด หลักการทำงานของ generics ใน Rust คือ monomorphization ซึ่งหมายถึงคอมไพเลอร์จะสร้างเวอร์ชันเฉพาะของฟังก์ชันหรือโครงสร้างสำหรับแต่ละประเภทที่ใช้จริงในเวลาคอมไพล์ ทำให้ไม่มีค่าใช้จ่ายในการรันไทม์เหมือนกับการใช้ไดนามิกดิสแพตช์ในภาษาอื่น ๆ

เรามาเริ่มกันด้วยฟังก์ชัน generic ที่หาค่ามากที่สุดในสไลซ์:

```rust
// ฟังก์ชัน generic ที่หาค่ามากที่สุดในสไลซ์ของประเภท T
// T ต้องนำ trait PartialOrd มาใช้เพื่อให้สามารถเปรียบเทียบได้
fn largest<T: PartialOrd + Copy>(list: &[T]) -> T {
    let mut largest = list[0];

    for &item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

// ตัวอย่างการใช้งาน
fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("จำนวนที่มากที่สุดคือ {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("ตัวอักษรที่มากที่สุดคือ {}", result);
}
```

ในตัวอย่างนี้เราได้กำหนด trait bound `T: PartialOrd + Copy` เพื่อให้แน่ใจว่าประเภท T สามารถเปรียบเทียบได้ด้วยoperators `>` และสามารถคัดลอกได้ด้วย `Copy` หากเราลบ `Copy` ออกเราจะต้องใช้การอ้างอิงแทน แต่ตัวอย่างนี้ทำให้เข้าใจได้ง่าย

ต่อไปเรามาดู struct generic กันบ้าง:

```rust
// จุดในมิติที่กำหนดโดยประเภท T
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    // เมธอดสร้างจุดใหม่
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }

    // เมธอดคืนค่าพิกัด x
    fn x(&self) -> &T {
        &self.x
    }
}

// ตัวอย่างการใช้งาน
fn main() {
    let integer = Point { x: 5, y: 10 };
    let float = Point { x: 1.0, y: 4.0 };

    println!("จุดจำนวนเต็ม: ({}, {})", integer.x(), integer.y());
    println!("จุดจุดทศนิยม: ({}, {})", float.x(), float.y());
}
```

ในตัวอย่างข้างต้น `Point` เป็น struct generic ที่สามารถเก็บพิกัดที่เป็นประเภทใดก็ได้ตราบใดที่ฟิลด์ `x` และ `y` มีประเภทเดียวกัน หากเราต้องการให้ `x` และ `y` เป็นประเภทที่แตกต่างกันเราสามารถใช้พารามิเตอร์แบบหลายตัวได้เช่น `struct Point<T, U> { x: T, y: U }`

สุดท้ายเรามาดู enum generic กันบ้าง เช่น `Option<T>` และ `Result<T, E>` ที่อยู่ในไลบรารีมาตรฐาน:

```rust
// การกำหนด enum Option ด้วย generic ทั่วไปในไลบรารีมาตรฐาน
// enum Option<T> {
//     Some(T),
//     None,
// }

// ตัวอย่างการใช้งาน Option<T>
fn main() {
    let some_number = Some(5);
    let some_string = Some(String::from("hello"));

    let absent_number: Option<i32> = None;
}

// การกำหนด enum Result ด้วย generic ทั่วไปในไลบรารีมาตรฐาน
// enum Result<T, E> {
//     Ok(T),
//     Err(E),
// }

// ตัวอย่างการใช้งาน Result<T, E>
fn divide(numerator: f64, denominator: f64) -> Result<f64, &'static str> {
    if denominator == 0.0 {
        Err("ไม่สามารถหารด้วยศูนย์ได้")
    } else {
        Ok(numerator / denominator)
    }
}

fn main() {
    match divide(10.0, 2.0) {
        Ok(result) => println!("ผลลัพธ์คือ {}", result),
        Err(e) => println!("ข้อผิดพลาด: {}", e),
    }
}
```

จากตัวอย่างเหล่านี้เราจะเห็นว่า generics ช่วยให้เราเขียนโค้ดที่เป็น generic ได้อย่างปลอดภัยและมีประสิทธิภาพ เพราะคอมไพเลอร์จะสร้างโค้ดเฉพาะเจาะจงสำหรับแต่ละประเภทที่ใช้จริง

## Trait bounds

Trait bounds คือเงื่อนไขที่เรากำหนดให้กับพารามิเตอร์ generic เพื่อให้แน่ใจว่าประเภทนั้นมีความสามารถบางอย่างที่เราต้องการ เราสามารถระบุหนึ่ง trait หรือหลาย trait ได้โดยใช้เครื่องหมาย `+` เพื่อเชื่อมต่อกัน นอกจากนี้เรายังสามารถใช้ `where` clause เพื่อทำให้เงื่อนไขอ่านง่ายขึ้นเมื่อมีหลายเงื่อนไขหรือมีความซับซ้อน

ตัวอย่างการใช้หลาย trait bounds พร้อมกัน:

```rust
use std::fmt::Display;

// ฟังก์ชันที่ต้องการให้ T นำทั้ง Display และ Clone มาใช้
fn print_and_clone<T: Display + Clone>(item: T) {
    println!("ค่าคือ: {}", item);
    let _copy = item.clone();
}
```

ในตัวอย่างนี้ `T` ต้องนำทั้ง `Display` (เพื่อให้สามารถพิมพ์ได้ด้วย `{}`) และ `Clone` (เพื่อให้สามารถคัดลอกได้) มาใช้ หากเราลบเงื่อนไขใดเงื่อนไขหนึ่งออกคอมไพเลอร์จะแจ้งข้อผิดพลาดเมื่อเราพยายามเรียกใช้เมธอดที่ต้องการเงื่อนไขนั้น

อีกวิธีหนึ่งคือการใช้ `where` clause ซึ่งมีประโยชน์เมื่อเรามีพารามิเตอร์หลายตัวหรือเงื่อนไขที่ยาวเหยียด:

```rust
use std::fmt::Display;

fn some_function<T, U>(t: T, u: U)
where
    T: Display + Clone,
    U: Clone + PartialEq,
{
    println!("t คือ: {}", t);
    let _t_clone = t.clone();
    let _u_clone = u.clone();
    // เปรียบเทียบ u กับตัวมันเองเพื่อแสดงว่า U ต้อง implement PartialEq
    let _ = u == u;
}
```

ที่นี่เราได้แยกเงื่อนไขออกเป็นบรรทัดต่าง ๆ ทำให้อ่านง่ายขึ้น นอกจากนี้เรายังสามารถใช้ `where` clause กับ impl block ได้อีกด้วย เพื่อกำหนด trait bounds สำหรับการ implement เมธอดหรือฟังก์ชันที่เกี่ยวข้องกับประเภทนั้น

ตัวอย่างการใช้ `where` กับ impl block:

```rust
struct Pair<T> {
    x: T,
    y: T,
}

impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}

// เราจะ implement เมธอด cmp_display เฉพาะเมื่อ T นำ Display และ PartialOrd มาใช้
impl<T> Pair<T>
where
    T: Display + PartialOrd,
{
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("สมาชิกที่มากที่สุดคือ x: {}", self.x);
        } else {
            println!("สมาชิกที่มากที่สุดคือ y: {}", self.y);
        }
    }
}
```

ในตัวอย่างนี้เมธอด `cmp_display` จะพร้อมใช้งานเฉพาะเมื่อประเภท `T` นำทั้ง `Display` และ `PartialOrd` มาใช้ ซึ่งช่วยให้เราสามารถเขียน implementation ที่มีเงื่อนไขเฉพาะเจาะจงได้โดยไม่ทำให้ signature ของ struct หรือ impl หลักซับซ้อนเกินไป

Trait bounds ยังสามารถใช้กับการ implement trait อื่น ๆ ได้อีกด้วย เช่นเมื่อเราต้องการให้ประเภทหนึ่งนำ trait `Copy` มาใช้ก็ต่อเมื่อองค์ประกอบภายในของมันก็นำ `Copy` มาใช้เช่นกัน นี่เป็นพื้นฐานของการอนุมาน trait ในไลบรารีมาตรฐาน เช่นการที่ `Option<T>` จะนำ `Clone` มาใช้เมื่อ `T` นำ `Clone` มาใช้

## Trait objects

บางครั้งเราต้องการเก็บประเภทที่แตกต่างกันซึ่งนำ trait เดียวกันมาใช้ในคอลเลกชันเดียวกัน เช่นเวกเตอร์ที่สามารถเก็บทั้ง `NewsArticle` และ `Tweet` ได้พร้อมกัน เพราะทั้งสองนำ trait `Summary` มาใช้ ในกรณีนี้เราไม่สามารถใช้ generics แบบปกติได้เพราะเวกเตอร์ต้องการประเภทที่แน่นอนในเวลาคอมไพล์ ดังนั้นเราจึงใช้ trait object ซึ่งทำโดยใช้คีย์เวิร์ด `dyn` ตามด้วยชื่อ trait ทำให้เราสามารถเก็บอินสแตนซ์ของประเภทต่าง ๆ ที่นำ trait นั้นมาใช้ได้ในชนิดเดียวกัน

ตัวอย่างการใช้ trait object กับ trait `Summary`:

```rust
use crate::{NewsArticle, Tweet, Summary};

fn main() {
    let article = NewsArticle {
        headline: String::from("Penguins win the Stanley Cup Championship!"),
        location: String::from("Pittsburgh, PA, USA"),
        author: String::from("Iceburgh"),
        content: String::from("The Pittsburgh Penguins once again are the best \
                                hockey team in the NHL."),
    };

    let tweet = Tweet {
        username: String::from("horse_ebooks"),
        content: String::from(
            "of course, as you probably already know, people",
        ),
        reply: false,
        retweet: false,
    };

    // สร้างเวกเตอร์ที่เก็บ trait object ของ Summary
    let items: Vec<Box<dyn Summary>> = vec![
        Box::new(article),
        Box::new(tweet),
    ];

    for item in items {
        println!("{}", item.summarize());
    }
}
```

ในตัวอย่างนี้เราได้สร้างเวกเตอร์ของ `Box<dyn Summary>` ซึ่งแต่ละองค์ประกอบคือกล่องที่ชี้ไปยังอินสแตนซ์ที่นำ trait `Summary` มาใช้ เมื่อเราเรียกเมธอด `summarize` ผ่าน trait object จะเกิดการส่งข้อความแบบไดนามิก (dynamic dispatch) หมายความว่าคอมไพเลอร์จะไม่ทราบว่าฟังก์ชันที่แท้จริงที่จะถูกเรียกคืออะไรจนถึงเวลารันไทม์ และจะใช้ตาราง vtable เพื่อค้นหาฟังก์ชันที่เหมาะสม

เรามาเปรียบเทียบระหว่าง static dispatch ที่เกิดจากการใช้ generics กับ dynamic dispatch ที่เกิดจากการใช้ trait object

เมื่อเราใช้ generics เช่นฟังก์ชัน `notify<T: Summary>(item: &T)` คอมไพเลอร์จะสร้างเวอร์ชันเฉพาะของฟังก์ชันสำหรับแต่ละประเภทที่ใช้เรียก เช่น `notify_NewsArticle` และ `notify_Tweet` ซึ่งทำให้เกิดการผูกมัดแบบคงที่ (static dispatch) และไม่มีค่าใช้จ่ายในการค้นหาฟังก์ชันเวลารันไทม์ แต่จะทำให้เกิดการคัดลอกโค้ด (code duplication) หากมีหลายประเภทที่ใช้ฟังก์ชันเดียวกันมากเกินไป

ในทางกลับกันเมื่อเราใช้ trait object เช่น `&dyn Summary` หรือ `Box<dyn Summary>` การเรียกเมธอดจะเกิดขึ้นผ่านไดนามิกดิสแพตช์ ซึ่งมีค่าใช้จ่ายเล็กน้อยจากการค้นหาฟังก์ชันใน vtable แต่ช่วยให้เราสามารถเก็บประเภทต่าง ๆ ที่นำ trait เดียวกันมาใช้ในคอลเลกชันเดียวกันได้ ซึ่งเป็นสิ่งที่ generics ทำไม่ได้โดยตรง

การเลือกระหว่าง static dispatch และ dynamic dispatch ขึ้นอยู่กับกรณีการใช้งาน หากเราทราบประเภททั้งหมดในเวลาคอมไพล์และต้องการประสิทธิภาพสูงสุด เราควรใช้ generics หากเราต้องการความยืดหยุ่นในการเก็บประเภทต่าง ๆ ร่วมกัน เราควรใช้ trait object

## Standard library traits

ไลบรารีมาตรฐานของ Rust มี trait หลายตัวที่ถูกออกแบบมาเพื่อให้พฤติกรรมทั่วไปที่เราต้องการบ่อย ๆ ทำให้เราไม่ต้องเขียนซ้ำเอง ทrait ที่สำคัญได้แก่ `Display`, `Debug`, `Clone`, `Copy`, `From`, `Into`, และ `Default` ซึ่งแต่ละตัวมีวัตถุประสงค์เฉพาะเจาะจงและมักถูกใช้ร่วมกับ derive macro เพื่อให้การ implement เป็นไปโดยอัตโนมัติ

เริ่มต้นด้วย `Display` และ `Debug` ซึ่งทั้งสองใช้สำหรับการแปลงค่าเป็นสตริง แต่มีวัตถุประสงค์แตกต่างกัน `Display` มีไว้สำหรับการแสดงผลต่อผู้ใช้ปลายทาง โดยใช้แมโคร `println!` หรือแมโคร `format!` กับเครื่องหมาย `{}` ในขณะที่ `Debug` มีไว้สำหรับนักพัฒนาในการดีบัก โดยใช้แมโคร `println!` กับเครื่องหมาย `{:?}` หรือ `{:#?}` เพื่อแสดงข้อมูลโครงสร้างอย่างละเอียด เราสามารถ implement ทั้งสอง trait ได้ด้วยตนเองหรือใช้ derive macro เมื่อเป็นไปได้

ตัวอย่างการ implement `Display` และ `Debug` ด้วยตนเองสำหรับโครงสร้าง `Point`:

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
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}

fn main() {
    let point = Point { x: 3, y: 4 };

    println!("จุด (Display): {}", point);
    println!("จุด (Debug): {:?}", point);
}
```

เมื่อรันโปรแกรมนี้เราจะเห็นว่า `Display` ให้ผลลัพธ์ในรูปแบบ `(3, 4)` ในขณะที่ `Debug` ให้ผลลัพธ์ในรูปแบบ `Point { x: 3, y: 4 }` ซึ่งแสดงฟิลด์ทั้งหมดอย่างชัดเจน

ต่อมาคือ `Clone` และ `Copy` ซึ่งทั้งสองใช้สำหรับการคัดลอกค่า แต่มีความแตกต่างกันอย่างสำคัญ `Clone` เป็น trait ที่ให้ความสามารถในการคัดลอกลึก (deep copy) ผ่านเมธอด `clone()` ซึ่งอาจมีการทำงานที่มีค่าใช้จ่ายสูงขึ้นอยู่กับประเภท ในขณะที่ `Copy` เป็น trait ที่บ่งบอกว่าประเภทสามารถคัดลอกได้โดยการคัดลอกบิตอย่างง่าย (shallow copy) โดยไม่มีค่าใช้จ่ายเพิ่มเติม และไม่ต้องใช้เมธอดใด ๆ ทั้งสิ้น ประเภทที่เป็น `Copy` จะต้องมีขนาดที่ทราบในเวลาคอมไพล์และไม่มีการจัดสรรทรัพยากรแบบไดนามิก เช่นตัวเลขพื้นฐาน boolean หรือ tuple ที่ประกอบด้วยประเภทเหล่านี้เท่านั้น

เราสามารถ derive ทั้งสอง trait ได้เมื่อเป็นไปได้ดังต่อไปนี้:

```rust
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let point1 = Point { x: 5, y: 10 };
    let point2 = point1.clone(); // ใช้ Clone
    let point3 = point1;         // ใช้ Copy เพราะ Point เป็น Copy

    println!("จุดที่ 1: {:?}", point1);
    println!("จุดที่ 2 (clone): {:?}", point2);
    println!("จุดที่ 3 (copy): {:?}", point3);
}
```

อย่างไรก็ตามหากโครงสร้างของเราประกอบด้วยประเภทที่ไม่เป็น `Copy` เช่น `String` หรือ `Vec<T>` เราจะไม่สามารถ derive `Copy` ได้ แต่เรายังสามารถ derive `Clone` ได้เพื่อให้สามารถทำการคัดลอกลึกได้

ต่อมาคือ `From` และ `Into` ซึ่งเป็น trait ที่ใช้สำหรับการแปลงประเภทหนึ่งไปยังอีกประเภทหนึ่ง `From<T>` ถูกใช้เมื่อเราต้องการกำหนดวิธีการแปลงจากประเภท `T` เป็นประเภทปัจจุบันของเรา ในขณะที่ `Into<T>` ถูกใช้เมื่อเราต้องการแปลงจากประเภทปัจจุบันของเราไปเป็นประเภท `T` โดยทั่วไปแล้วหากเรา implement `From<T>` สำหรับประเภทหนึ่ง เราจะทำให้เราได้ `Into<ที่นั้น>` โดยอัตโนมัติผ่านการ implement ทั่วไปในไลบรารีมาตรฐาน ดังนั้นเรามักจะ implement เพียงฝั่งเดียวเท่านั้น

ตัวอย่างการใช้ `From` และ `Into` กับโครงสร้าง `Point` ที่แปลงจาก tuple:

```rust
struct Point {
    x: i32,
    y: i32,
}

// Implement From<(i32, i32)> สำหรับ Point
impl From<(i32, i32)> for Point {
    fn from((x, y): (i32, i32)) -> Self {
        Self { x, y }
    }
}

fn main() {
    let point_from_tuple: Point = (5, 10).into(); // ใช้ Into ที่ได้จาก From
    println!("จุดจากทูเปิล: ({}, {})", point_from_tuple.x, point_from_tuple.y);

    // หรือใช้ From โดยตรง
    let point_direct = Point::from((3, 7));
    println!("จุดจาก from โดยตรง: ({}, {})", point_direct.x, point_direct.y);
}
```

ในตัวอย่างนี้เราได้ implement `From<(i32, i32)>` สำหรับ `Point` ซึ่งทำให้เราสามารถใช้ `.into()` บนทูเปิลเพื่อแปลงเป็น `Point` ได้อย่างสะดวก และเนื่องจากมีการ implement `From` อยู่แล้ว เราจึงสามารถใช้ `Point::from` ได้โดยตรงเช่นกัน

สุดท้ายคือ `Default` ซึ่งให้ค่าเริ่มต้นสำหรับประเภทผ่านฟังก์ชัน `default()` นี่เป็นประโยชน์เมื่อเราต้องการสร้างอินสแตนซ์โดยไม่ต้องระบุค่าเริ่มต้นทุกฟิลด์ โดยเฉพาะอย่างยิ่งเมื่อทำงานกับสตรักต์ที่มีหลายฟิลด์ เราสามารถ implement `Default` ด้วยตนเองหรือใช้ derive เมื่อเป็นไปได้

ตัวอย่างการใช้ `Default` กับสตรักต์ที่มีค่าเริ่มต้น:

```rust
#[derive(Default, Debug)]
struct Settings {
    width: u32,
    height: u32,
    enabled: bool,
}

fn main() {
    // ใช้ Default เพื่อสร้างอินสแตนซ์ด้วยค่าเริ่มต้น
    let settings = Settings::default();
    println!("การตั้งค่าเริ่มต้น: {:?}", settings);

    // กำหนดค่าเฉพาะบางฟิลด์
    let custom_settings = Settings {
        width: 800,
        height: 600,
        ..Default::default() // ใช้ค่าเริ่มต้นสำหรับฟิลด์ที่เหลือ
    };
    println!("การตั้งค่ากำหนดเอง: {:?}", custom_settings);
}
```

จากตัวอย่างข้างต้นเราเห็นว่า `default()` จะคืนค่าที่เป็นศูนย์หรือค่าเริ่มต้นตามประเภทของฟิลด์แต่ละฟิลด์ และเราสามารถใช้โครงสร้างอัปเดตกับ `..Default::default()` เพื่อกำหนดค่าเฉพาะบางฟิลด์และให้ฟิลด์ที่เหลือใช้ค่าเริ่มต้นได้

 trait เหล่านี้ไม่เพียงแต่ช่วยให้เราเขียนโค้ดได้สั้นลงเท่านั้น แต่ยังช่วยให้เกิดความสอดคล้องและความสามารถในการทำงานร่วมกันกับไลบรารีมาตรฐานและไลบรารีของบุคคลที่สามอีกด้วย ตัวอย่างเช่นฟังก์ชันทั่วไปในไลบรารีมาตรฐานมักจะกำหนดขอบเขตของพารามิเตอร์ด้วย trait เหล่านี้ เช่น `FromStr` สำหรับการแปลงจากสตริง หรือ `Hash` สำหรับการใช้งานในแฮชแมป

## เชื่อมโยงกับ rs-wsProxy

ในโปรเจกต์จริงอย่าง rs-wsProxy ซึ่งเป็นพร็อกซี่ WebSocket ที่สร้างด้วย Rust เราจะเห็นการใช้ trait และ generics อย่างแพร่หลายในหลายไลบรารีที่เราพึ่งพา เช่น Axum ซึ่งเป็นเฟรมเวิร์กเว็บที่ใช้สำหรับสร้างเซิร์ฟเวอร์ HTTP และ WebSocket ใน Axum แนวคิดของ trait มีความสำคัญอย่างมากผ่าน trait เช่น `IntoResponse` ซึ่งใช้แปลงค่าต่าง ๆ เป็นการตอบสนอง HTTP ได้อย่างยืดหยุ่น ตัวอย่างเช่นเราสามารถคืนค่าเป็น `String`, `&str`, หรือแม้แต่ประเภทที่เราสร้างเองที่ได้ implement `IntoResponse` เพื่อให้ Axum รู้ว่าจะแปลงค่านั้นเป็นการตอบสนอง HTTP อย่างไรได้อย่างถูกต้อง

อีกตัวอย่างหนึ่งคือ `FromRequest` ซึ่งใช้สกัดข้อมูลจากคำขอ HTTP เช่นหัวข้อ คิวรีสตริง หรือร่างกายของคำขอ เพื่อส่งต่อให้กับแฮนด์เลอร์ของเรา ด้วยการ implement `FromRequest` สำหรับประเภทของเราเอง เราสามารถดึงข้อมูลที่ต้องการจากคำขอได้อย่างปลอดภัยและเป็นธรรมชาติ เช่นการดึงโทเค็นการตรวจสอบสิทธิ์จากหัวข้อ Authorization หรือการอ่านข้อมูล JSON จากร่างกายของคำขอ

นอกจากนี้ `FromRef` และ `FromRefMut` ยังใช้ในการแชร์สถานะหรือการเชื่อมต่อระหว่างชั้นต่าง ๆ ของแอปพลิเคชัน เช่นการแชร์การเชื่อมต่อฐานข้อมูลหรือการกำหนดค่าผ่านสถานะของแอปพลิเคชัน ซึ่งช่วยให้เราหลีกเลี่ยงการใช้ตัวแปรทั่วไปและทำให้โค้ดมีความปลอดภัยสูงขึ้นในบริบทของการทำงานแบบอะซิงโครนัส

ในส่วนของไลบรารี clap ซึ่งใช้สำหรับสร้างอินเตอร์เฟซบรรทัดคำสั่ง เราจะเห็นการใช้ derive macro เพื่อสร้างการ implement trait `Parser` จากโครงสร้างที่เรากำหนดเอง ทำให้เราสามารถกำหนดอาร์กิวเมนต์และออปชันของคำสั่งได้โดยการกำหนดฟิลด์ในสตรักต์และใส่แอตทริบิวต์ derive ที่เหมาะสม เช่น `#[arg(long)]` หรือ `#[arg(short, long)]` จากนั้น clap จะสร้างโค้ดที่จำเป็นในการวิเคราะห์อาร์กิวเมนต์บรรทัดคำสั่งและเติมค่าให้กับฟิลด์ของสตรักต์โดยอัตโนมัติ ซึ่งช่วยลดการเขียนโค้ดซ้ำซ้อนและทำให้การบำรุงรักษาง่ายขึ้น

สุดท้ายในด้านการเขียนโปรแกรมแบบอะซิงโครนัส Rust ใช้ trait `Future` เป็นแกนหลักของการทำงานแบบ non-blocking ฟิวเจอร์แสดงถึงการคำนวณที่จะเสร็จสิ้นในอนาคตและสามารถรอคอยได้โดยใช้ `.await` คีย์เวิร์ด ไลบรารีอย่าง Tokio ให้รันไทม์ที่สามารถรันฟิวเจอร์เหล่านี้ได้อย่างมีประสิทธิภาพ โดยใช้ระบบจัดการงานและการแจ้งเตือนเหตุการณ์ (event loop) ที่ทำให้เราสามารถเขียนโค้ดแบบอะซิงโครนัสที่ดูเหมือนซิงโครนัสได้โดยไม่บล็อกเธรดหลัก

การทำความเข้าใจว่า trait และ generics ทำงานอย่างไรในระดับพื้นฐานจะช่วยให้เราสามารถอ่านและเขียนโค้ดที่ใช้ไลบรารีเหล่านี้ได้อย่างลึกซึ้งมากขึ้น เราจะเห็นว่าการออกแบบไลบรารีเหล่านี้ไม่ใช่เพียงแค่การให้ฟังก์ชันสำเร็จรูปเท่านั้น แต่ยังเป็นการให้กรอบการทำงานที่เราสามารถขยายและปรับแต่งได้ผ่านทาง trait bounds และ generics ซึ่งเป็นหัวใจสำคัญของการเขียนโค้ดที่เป็นนามธรรมและสามารถนำกลับมาใช้ใหม่ได้ใน Rust

## สรุป

ในบทนี้เราได้สำรวจแนวคิดหลักสองประการของ Rust ที่ทำให้ภาษานี้มีพลังและความปลอดภัยสูง ได้แก่ traits และ generics เราได้เห็นว่า trait ทำหน้าที่เหมือนกับ interface ในภาษาอื่น ๆ แต่มีความยืดหยุ่นมากกว่าด้วยความสามารถในการให้ default implementation และอนุญาตให้เรา implement trait สำหรับประเภทใด ๆ ได้ไม่ว่าจะเป็นสตรักต์ที่เราสร้างเองหรือประเภทจากไลบรารีมาตรฐาน เราได้เรียนรู้วิธีการกำหนด trait การ implement trait ให้กับประเภทต่าง ๆ และการใช้ trait เป็นพารามิเตอร์ในฟังก์ชันทั้งในรูปแบบ `impl Trait` และรูปแบบ trait bound ปกติ พร้อมทั้งการใช้หลาย trait bounds และ `where` clause เพื่อจัดการเงื่อนไขที่ซับซ้อน

ในส่วนของ generics เราได้เห็นว่ามันช่วยให้เราเขียนฟังก์ชัน สตรักต์ และเอ็นัมที่ทำงานกับหลายประเภทได้โดยไม่ต้องทำซ้ำโค้ด ผ่านกลไกการสร้างโค้ดเฉพาะเจาะจงในเวลาคอมไพล์ (monomorphization) ซึ่งทำให้เราได้ประสิทธิภาพเทียบเท่ากับการเขียนโค้ดเฉพาะเจาะจงแต่ไม่ต้องเขียนซ้ำ เราได้ดูตัวอย่างของฟังก์ชัน generic ที่หาค่ามากที่สุดในสไลซ์ สตรักต์ generic เช่น `Point<T>` และเอ็นัม generic เช่น `Option<T>` และ `Result<T, E>` ที่อยู่ในไลบรารีมาตรฐาน

เราได้พูดถึง trait bounds ซึ่งเป็นเงื่อนไขที่เรากำหนดให้กับพารามิเตอร์ generic เพื่อให้แน่ใจว่าประเภทนั้นมีความสามารถบางอย่างที่เราต้องการ เช่นการสามารถเปรียบเทียบได้ (`PartialOrd`) หรือการสามารถแสดงผลได้ (`Display`) เราได้เห็นวิธีการรวมหลาย trait bounds ด้วยเครื่องหมาย `+` และการใช้ `where` clause เพื่อให้เงื่อนไขอ่านง่ายขึ้นเมื่อมีหลายเงื่อนไขหรือมีความยาวเหยียด

ในส่วนของ trait objects เราได้เรียนรู้วิธีการใช้ `dyn Trait` เพื่อเก็บอินสแตนซ์ของประเภทต่าง ๆ ที่นำ trait เดียวกันมาใช้ในคอลเลกชันเดียวกัน ผ่านกลไกไดนามิกดิสแพตช์ ซึ่งมีค่าใช้จ่ายเล็กน้อยจากการค้นหาฟังก์ชันใน vtable แต่ช่วยให้เราสามารถเขียนโค้ดที่ยืดหยุ่นได้เมื่อเราไม่ทราบประเภทที่แน่นอนในเวลาคอมไพล์ เราได้เปรียบเทียบระหว่าง static dispatch ที่เกิดจากการใช้ generics กับ dynamic dispatch ที่เกิดจากการใช้ trait objects และได้อภิปรายถึงสถานการณ์ที่ควรใช้แต่ละแบบ

สุดท้ายเราได้สำรวจ trait ที่สำคัญในไลบรารีมาตรฐาน เช่น `Display`, `Debug`, `Clone`, `Copy`, `From`, `Into`, และ `Default` พร้อมทั้งตัวอย่างการ implement และการใช้ derive macro เพื่อลดภาระในการเขียนโค้ดซ้ำซ้อน เราได้เห็นว่าทrait เหล่านี้ไม่เพียงแต่ให้พฤติกรรมพื้นฐานเท่านั้น แต่ยังเป็นพื้นฐานสำหรับการทำงานร่วมกับไลบรารีอื่น ๆ เช่น Axum, clap และ Tokio ในโปรเจกต์จริงอย่าง rs-wsProxy

โดยสรุปแล้ว traits และ generics คือรากฐานที่ทำให้ Rust สามารถให้ความปลอดภัยของประเภทและการทำงานที่มีประสิทธิภาพสูงได้ในขณะที่ยังคงความสามารถในการแสดงออกและการนำกลับมาใช้ใหม่ การทำความเข้าใจแนวคิดเหล่านี้อย่างลึกซึ้งจะช่วยให้เราเขียนโค้ด Rust ที่แข็งแกร่ง ยืดหยุ่น และสามารถบำรุงรักษาได้ไม่ว่าจะเป็นโครงการขนาดเล็กหรือระบบขนาดใหญ่ที่ซับซ้อน

สำหรับผู้ที่ต้องการศึกษาต่อไป บทต่อไปในซีรีส์นี้จะพูดถึงการเขียนโปรแกรมแบบอะซิงโครนัสใน Rust โดยใช้ไลบรารี Tokio ซึ่งเราจะได้เห็นว่า trait `Future` และระบบ async/await ทำงานร่วมกันอย่างไรเพื่อให้เราสามารถเขียนโค้ดที่ไม่บล็อกและมีประสิทธิภาพสูงได้

ลิงก์ไปยังบทก่อนหน้า: [Collections, Iterators และ Error Handling](/posts/rust/rust-collections-errors/)  
ลิงก์ไปยังบทถัดไป: [Async Rust กับ Tokio](/posts/rust/rust-async-tokio/)