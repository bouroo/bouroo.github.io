---
title: "Async Rust กับ Tokio"
subtitle: ""
date: 2026-07-12T09:00:00+07:00
lastmod: 2026-07-12T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: " async/await, Tokio runtime, spawning tasks, channels และ async I/O สำหรับเขียน network applications"
license: ""
images: []
tags: ["Rust", "Tutorial", "Tokio"]
categories: ["Rust"]
featuredImage: "featured-image.jpeg"
featuredImagePreview: "featured-image.jpeg"
lightgallery: true
---

<!--more-->

## บทนำ: Async คืออะไรและทำไมถึงสำคัญ

การเขียนโปรแกรมแบบ asynchronous (async) เป็นแนวคิดที่ช่วยให้เราสามารถเขียนโค้ดที่ทำงานหลายอย่างพร้อมกันได้โดยไม่ต้องสร้าง thread จำนวนมากๆ ซึ่งเป็นสิ่งสำคัญอย่างยิ่งสำหรับแอปพลิเคชันประเภท network applications เช่นเว็บเซิร์ฟเวอร์, proxy server, หรือไคลเอนต์ที่ต้องจัดการกับการเชื่อมต่อเครือข่ายจำนวนมากพร้อมกัน

ในภาษา Rust แนวคิดของ async ไม่ได้ถูกฝังมาอยู่ในภาษาโดยตรงเหมือนในบางภาษา เช่น Go หรือ JavaScript แต่ Rust ให้เครื่องมือพื้นฐาน (primitives) สำหรับการเขียนโค้ดแบบ async ผ่าน `async/await` syntax และปล่อยให้ผู้พัฒนาเลือกใช้ runtime ที่เหมาะสมกับความต้องการของตนเอง ซึ่งหนึ่งใน runtime ที่ได้รับความนิยมสูงสุดคือ **Tokio**

ในบทความก่อนหน้านี้ (ภาค 5: Traits และ Generics) เราได้พูดถึงแนวคิดพื้นฐานของ Rust ที่ช่วยให้เราเขียนโค้ดที่ทั่วถึงและปลอดภัยได้ บทความนี้เราจะต่อยอดด้วยการดูว่าเราจะใช้แนวคิดเหล่านั้นร่วมกับ async/await และ Tokio เพื่อสร้าง network application ที่มีประสิทธิภาพสูงได้อย่างไร

## ทำไมต้องใช้ Async แทน Thread-per-Connection?

ในแบบจำลองการเขียนโปรแกรมแบบดั้งเดิมสำหรับ network server เรามักจะสร้าง thread หนึ่งตัวขึ้นมาสำหรับแต่ละการเชื่อมต่อที่เข้ามา (thread-per-connection model) วิธีนี้ทำงานได้ดีเมื่อจำนวนการเชื่อมต่อไม่สูงมาก แต่เมื่อจำนวนการเชื่อมต่อเพิ่มขึ้นเป็นหลักหมื่นหรือหลักแสน การสร้าง thread จำนวนมากจะกินทรัพยากรระบบอย่างมาก เนื่องจากแต่ละ thread มี stack ของตัวเอง (โดยปกติแล้ว 1MB หรือมากกว่า) และการสลับบริบทระหว่าง thread (context switching) มีค่าใช้จ่ายสูง

ในทางตรงกันข้าม โมเดลแบบ asynchronous ใช้แนวคิดของ event loop และ lightweight tasks (บางครั้งเรียกว่า green threads หรือ coroutines) ซึ่งทำงานอยู่บน thread จำนวนน้อย (เช่น จำนวน CPU cores) แต่ละ task จะถูกระงับ (suspended) เมื่อมันกำลังรอ I/O operation เช่นการอ่านข้อมูลจาก socket หรือการเขียนไฟล์ ทำให้ thread นั้นสามารถไปทำงานอื่นๆ ทำงานอื่นที่พร้อมจะทำงานได้แทน ทำให้เราสามารถจัดการกับการเชื่อมต่อจำนวนมหาศาลได้ด้วยทรัพยากรที่น้อยกว่ามาก

ใน Rust เราไม่มี runtime async ในตัวภาษา ดังนั้นเราจึงต้องเลือกใช้ external crate เช่น `Tokio` หรือ `async-std` เพื่อให้ได้ runtime ที่พร้อมใช้งาน Tokio ได้กลายเป็นมาตรฐานเด facto สำหรับการเขียน network application ใน Rust เนื่องจากประสิทธิภาพสูง ฟีเจอร์ครบถ้วน และเอกสารที่ดี

## Async/Await ใน Rust

ใน Rust เราใช้คำสั่ง `async` เพื่อกำหนดฟังก์ชันที่สามารถทำงานแบบ asynchronous ได้ เมื่อเราประกาศฟังก์ชันด้วย `async fn` คอมไพเลอร์จะแปลงฟังก์ชันนั้นให้กลายเป็นฟังก์ชันที่คืนค่าเป็น `Future` ซึ่งเป็น trait ที่แสดงถึงการคำนวณที่อาจจะยังไม่เสร็จสิ้นในทันที

ตัวอย่างฟังก์ชัน async ง่ายๆ:

```rust
async fn fetch_data() -> String {
    // จำลองการทำงานที่ใช้เวลา เช่น การดึงข้อมูลจากเครือข่าย
    // ในความเป็นจริงเราอาจใช้ tokio::time::sleep หรือ tokio::net::TcpStream ที่นี่
    // แต่เพื่อความเรียบง่ายเราจะใช้ std::thread::sleep ซึ่งเป็น blocking! 
    // ในการใช้งานจริงเราควรใช้ async sleep จาก tokio
    // แต่เพื่อแสดงให้เห็นโครงสร้าง เราจะใช้ blocking ชั่วคราว (ไม่แนะนำในโค้ดจริง)
    std::thread::sleep(std::time::Duration::from_secs(1));
    "data from server".to_string()
}
```

อย่างไรก็ตาม ตัวอย่างข้างต้นใช้ `std::thread::sleep` ซึ่งเป็น blocking call และไม่ควรใช้ในโค้ด async จริงๆ เราควรใช้ `tokio::time::sleep` แทน แต่เราจะแสดงตัวอย่างที่ถูกต้องในภายหลัง

เมื่อเรามี `Future` แล้ว เราใช้คำสั่ง `.await` เพื่อรอให้ future นั้นเสร็จสิ้นและได้ผลลัพธ์ออกมา ตัวอย่างการใช้งาน:

```rust
async fn process_data() -> String {
    let data = fetch_data().await; // รอจนกว่า fetch_data จะเสร็จ
    format!("Processing: {}", data)
}
```

นอกจาก `async fn` แล้ว Rust ยังอนุญาตให้เราใช้ `async block` ได้อีกด้วย:

```rust
let future = async {
    let data = fetch_data().await;
    format!("Processing: {}", data)
};

// เมื่อเราพร้อมที่จะรอผลลัพธ์:
// let result = future.await;
```

สิ่งสำคัญที่ต้องจำไว้คือ `async fn` หรือ `async block` เพียงแค่สร้าง `Future` ขึ้นมาเท่านั้น มันจะไม่เริ่มทำงานจนกว่าเราจะเรียก `.await` บนมัน ซึ่งแตกต่างจากภาษาบางภาษาที่การเรียกฟังก์ชัน async จะเริ่มทำงานทันที

## Tokio Runtime: หัวใจของการทำงาน Async

ใน Rust เราต้องมี runtime เพื่อที่จะรัน `Future` ของเรา Tokio เป็นหนึ่งใน runtime ที่ได้รับความนิยมสูงสุด มันให้ฟีเจอร์ครบครันสำหรับการเขียน network application รวมถึง TCP/UDP sockets, timers, สัญญาณ (signals), และ synchronization primitive ต่างๆ

วิธีที่ง่ายที่สุดในการเริ่มต้นใช้งาน Tokio คือการใช้ attribute macro `#[tokio::main]` บนฟังก์ชัน `main`:

```rust
use tokio;

#[tokio::main]
async fn main() {
    println!("Hello from Tokio!");
    // ที่นี่เราสามารถเรียกใช้ async function ได้โดยตรง
    let data = fetch_data().await;
    println!("{}", data);
}
```

แมคโครร `#[tokio::main]` จะสร้าง runtime แบบ multi-threaded ตามจำนวน CPU cores ของระบบโดยอัตโนมัติ และรันฟังก์ชัน `main` ของเราบน runtime นั้น

หากเราต้องการควบคุมการตั้งค่ารายละเอียดของ runtime เราสามารถใช้ `tokio::runtime::Builder` ได้โดยตรง:

```rust
use tokio::runtime;

fn main() {
    // สร้าง runtime แบบ multi-threaded ที่มี 4 worker threads
    let runtime = runtime::Builder::new_multi_thread()
        .worker_threads(4)
        .enable_all() // เปิดฟีเจอร์ทั้งหมดที่จำเป็นสำหรับ net, time, ฯลฯ
        .build()
        .expect("Failed to build runtime");

    // รัน future บน runtime ที่เราสร้างขึ้น
    runtime.block_on(async {
        println!("Hello from custom Tokio runtime!");
        let data = fetch_data().await;
        println!("{}", data);
    });
}
```

เราสามารถเลือกใช้ runtime แบบ single-threaded ได้โดยใช้ `Builder::new_current_thread()` ซึ่งเหมาะสำหรับแอปพลิเคชันที่ไม่ต้องการความparallelism มากนัก หรือต้องการหลีกเลี่ยง overhead จากการสลับบริบทระหว่าง thread

## การสร้าง Task ด้วย Tokio

ใน Tokio เราไม่ได้สร้าง thread โดยตรง แต่เราสร้าง "task" ซึ่งเป็นหน่วยการทำงานที่เบากว่า thread มาก และถูกจัดการโดย Tokio runtime เราใช้ฟังก์ชัน `tokio::spawn` เพื่อสร้าง task ใหม่:

```rust
#[tokio::main]
async fn main() {
    // สร้าง task ใหม่ที่จะทำงานพร้อมกับ main task
    let handle = tokio::spawn(async {
        println!("Running in a spawned task!");
        // ทำบางอย่างที่ใช้เวลา เช่น รอเวลา
        tokio::time::sleep(tokio::time::Duration::from_secs(2)).await;
        println!("Task completed!
    });

    // รอให้ task ที่เราสร้างเสร็จสิ้น
    // handle เป็นประเภท JoinHandle ที่ช่วยให้เราสามารถรอผลลัพธ์ได้
    let result = handle.await;
    println!("Task finished with result: {:?}", result);
}
```

ความแตกต่างระหว่าง task กับ thread:
- Task น้ำหนักเบากว่า thread มาก (มักจะใช้หน่วยความจำเพียงไม่กี่ร้อยไบต์)
- การสลับบริบทระหว่าง task ทำโดย runtime ไม่ใช่โดย kernel ดังนั้นจึงเร็วกว่า
- Task ทั้งหมดทำงานบนชุด thread เดียวกันที่ถูกจัดการโดย runtime (โดยทั่วไปคือจำนวน CPU cores)

เมื่อเราเรียก `tokio::spawn` เราจะได้คืนค่าเป็น `JoinHandle<T>` โดยที่ `T` คือประเภทของค่าที่ task นั้นคืนกลับมาเมื่อเสร็จสิ้น เราสามารถใช้ `.await` บน `JoinHandle` เพื่อรอให้ task เสร็จและรับค่าที่มันคืนมาได้

สิ่งสำคัญที่ต้องจำไว้เมื่อใช้ `tokio::spawn` คือการใช้ `move` closure หากเราต้องการย้ายค่าจากภายนอกเข้าไปใน task:

```rust
#[tokio::main]
async fn main() {
    let data = vec![1, 2, 3, 4, 5];
    let handle = tokio::spawn(move || async {
        // `data` ถูกย้ายเข้าไปใน task นี้ด้วย move
        let sum: i32 = data.iter().sum();
        println!("Sum: {}", sum);
        sum
    });

    let result = handle.await.unwrap();
    println!("Result from task: {}", result);
}
```

หากเราลืม `move` คอมไพเลอร์จะแจ้งข้อผิดพลาดเพราะ `data` ถูกยืมโดย reference แต่ task ที่ถูกสปาวน์อาจมีอายุยาวนานกว่าฟังก์ชันปัจจุบัน

## ช่องทางสื่อสารระหว่าง Task: Channels

เมื่อเรามีหลาย task ที่ต้องการสื่อสารกัน เราสามารถใช้ช่องทาง (channels) ที่ให้โดย Tokio ได้ โดยเฉพาะอย่างยิ่ง `mpsc` (multiple producer, single consumer) ช่วยให้เราสามารถส่งข้อความจากหลาย task ไปยัง task เดียวได้อย่างปลอดภัย

ตัวอย่างการใช้ `mpsc` channel:

```rust
use tokio::sync::mpsc;
use tokio::time;

#[tokio::main]
async fn main() {
    // สร้าง channel: tx สำหรับส่ง, rx สำหรับรับ
    let (tx, mut rx) = mpsc::channel(32); // 32 คือขนาด buffer

    // สร้างหลาย task ที่ส่งข้อความไปยังตัวรับเดียวกัน
    for i in 0..3 {
        let tx = tx.clone(); // คลอน tx เพื่อให้แต่ละ task มีสำเนาของตัวส่ง
        tokio::spawn(async move {
            // แต่ละ task ส่งข้อความหลายครั้ง
            for j in 0..3 {
                let msg = format!("task {} message {}", i, j);
                if tx.send(msg).await.is_err() {
                    // ตัวรับได้ยกเลิกการทำงานแล้ว
                    eprintln!("Receiver dropped, stopping");
                    break;
                }
                // หน่วงเวลาเล็กน้อยเพื่อให้เห็นการสลับกันทำงาน
                time::sleep(time::Duration::from_millis(100)).await;
            }
        });
    }

    // ตัวรับทำงานใน main task
    // เราจะรับข้อความทั้งหมดแล้วพิมพ์ออกมา
    while let Some(msg) = rx.recv().await {
        println!("Received: {}", msg);
    }
    println!("All messages received.");
}
```

นอกจาก `mpsc` แล้ว Tokio ยังมี `oneshot` channel สำหรับการส่งข้อความเพียงครั้งเดียวจากผู้ส่งหนึ่งไปยังผู้รับหนึ่ง ซึ่งเหมาะสำหรับการส่งสัญญาณว่า task หนึ่งได้เสร็จสิ้นงานแล้ว

ตัวอย่างการใช้ `oneshot`:

```rust
use tokio::sync::oneshot;
use tokio::time;

#[tokio::main]
async fn main() {
    // สร้าง oneshot channel
    let (tx, rx) = oneshot::channel::<i32>();

    // ส่งค่าจาก task หนึ่ง
    tokio::spawn(async move {
        // ทำบางอย่างที่ใช้เวลา
        time::sleep(time::Duration::from_secs(1)).await;
        // ส่งผลลัพธ์
        let _ = tx.send(42);
    });

    // รอรับผลลัพธ์ใน main task
    match rx.await {
        Ok(value) => println!("Received: {}", value),
        Err(_) => eprintln!("Sender dropped without sending a value"),
    }
}
```

## Async I/O กับ Tokio

Tokio ให้การสนับสนุนการทำ I/O แบบ asynchronous ผ่าน trait เช่น `AsyncReadExt` และ `AsyncWriteExt` ซึ่งเป็นส่วนขยายของ `tokio::io::AsyncRead` และ `tokio::io::AsyncWrite` ตามลำดับ ทำให้เราสามารถใช้เมธอดเช่น `.read()`, `.write()`, `.shutdown()` ได้ในรูปแบบ async

ตัวอย่างการเขียน Echo Server แบบง่ายๆ ด้วย Tokio:

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> io::Result<()> {
    // ผูกที่อยู่และพอร์ต
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Server listening on 127.0.0.1:8080");

    loop {
        // รอการเชื่อมต่อใหม่เข้ามา (นี้คือ async operation)
        let (socket, addr) = listener.accept().await?;
        println!("Accepted connection from: {}", addr);

        // สำหรับแต่ละการเชื่อมต่อ เราสร้าง task ใหม่เพื่อจัดการมัน
        tokio::spawn(async move {
            if let Err(e) = process_socket(socket).await {
                eprintln!("Error processing socket: {}", e);
            }
        });
    }
}

async fn process_socket(mut socket: TcpStream) -> io::Result<()> {
    let mut buffer = [0; 1024];

    loop {
        // อ่านข้อมูลจาก socket (async)
        let n = match socket.read(&mut buffer).await {
            // ปิดการเชื่อมต่อเมื่ออ่านไม่ได้ข้อมูล (client ปิด connection)
            Ok(0) => return Ok(()),
            Ok(n) => n,
            Err(e) => return Err(e),
        };

        // ส่งข้อมูลที่อ่านได้กลับไปยัง client (echo)
        socket.write_all(&buffer[..n]).await?;
    }
}
```

ในตัวอย่างข้างต้น:
1. เราสร้าง `TcpListener` ที่ฟังอยู่ที่พอร์ต 8080
2. ในลูปไม่สิ้นสุด เราเรียก `listener.accept().await` เพื่อรอการเชื่อมต่อใหม่เข้ามาโดยไม่บล็อก thread
3. เมื่อมีการเชื่อมต่อใหม่ เราสร้าง task ใหม่ด้วย `tokio::spawn` เพื่อจัดการกับการเชื่อมต่อนั้นโดยเฉพาะ ทำให้เราสามารถจัดการการเชื่อมต่อหลายๆ รายการได้พร้อมกันโดยไม่บล็อกกัน
4. ในแต่ละ task เราอ่านข้อมูลจาก socket ด้วย `socket.read().await` และเขียนกลับไปด้วย `socket.write_all().await` ซึ่งทั้งสองการทำงานนี้เป็นแบบ non-blocking เนื่องจากเราใช้ `.await`

## การใช้ `tokio::select!` เพื่อรอหลาย Future พร้อมกัน

บางครั้งเราต้องการรอให้หนึ่งในหลายๆ future เสร็จสิ้นก่อน แล้วจึงดำเนินการต่อ ตัวอย่างเช่น เราอาจต้องการรอทั้งการรับข้อความจาก channel และการหมดเวลา timeout พร้อมกัน และดำเนินการตามอย่างใดอย่างหนึ่งที่เกิดขึ้นก่อน

Tokio ให้แมโคร `select!` มาเพื่อวัตถุประสงค์นี้โดยเฉพาะ มันทำงานคล้ายกับ `match` แต่สำหรับ future โดยจะทำงานกับตัวแรกที่พร้อม

ตัวอย่างการใช้ `select!` พร้อม timeout:

```rust
use tokio::time::{self, Duration};

#[tokio::main]
async fn main() {
    // สร้าง future ที่จะเสร็จหลังจาก 2 วินาที
    let delay = time::sleep(Duration::from_secs(2));

    // สร้าง future ที่จะเสร็จทันทีด้วยค่า "hello"
    let message = async { "hello" };

    // ใช้ select! เพื่อรอให้ใดๆ ของสอง future นี้เสร็จก่อน
    tokio::select! {
        msg = message => {
            println!("Received message: {}", msg);
        }
        _ = delay => {
            println!("Time is up!");
        }
    }
}
```

ในตัวอย่างนี้ ข้อความ "hello" จะถูกพิมพ์ออกมาก่อนเพราะ future ของ message เสร็จเร็วกว่า delay 2 วินาที

ตัวอย่างที่เป็นประโยชน์มากขึ้นคือการใช้ `select!` ในการทำ bidirectional communication ระหว่าง WebSocket และ TCP socket เช่นในโปรเจกต์ wsProxy ของเรา:

```rust
// สมมติว่าเรามี ws_stream และ tcp_stream
tokio::select! {
    // รับข้อความจาก WebSocket และส่งไปยัง TCP
    result = ws_stream.next() => {
        match result {
            Some(Ok(msg)) => {
                if let Err(e) = tcp_stream.write_all(&msg.into()).await {
                    eprintln!("Failed to write to TCP: {}", e);
                    break;
                }
            }
            Some(Err(e)) => {
                eprintln!("WebSocket error: {}", e);
                break;
            }
            None => {
                // WebSocket ปิดการเชื่อมต่อ
                break;
            }
        }
    }
    // รับข้อมูลจาก TCP และส่งไปยัง WebSocket
    result = tcp_stream.read(&mut buf) => {
        match result {
            Ok(0) => {
                // TCP ปิดการเชื่อมต่อ
                break;
            }
            Ok(n) => {
                if let Err(e) = ws_stream.send(web_socket::Message::Binary(
                    buf[..n].to_vec()
                )).await {
                    eprintln!("Failed to send to WebSocket: {}", e);
                    break;
                }
            }
            Err(e) => {
                eprintln!("Failed to read from TCP: {}", e);
                break;
            }
        }
    }
}
```

โครงสร้าง `select!` นี้ช่วยให้เราสามารถจัดการกับการไหลของข้อมูลสองทางได้อย่างมีประสิทธิภาพ โดยไม่ต้องบล็อกที่การอ่านจากทางใดทางหนึ่ง

## การปิดการทำงานอย่างสงบ (Graceful Shutdown)

ในแอปพลิเคชันเซิร์ฟเวอร์ที่ดี เราต้องการให้มันสามารถปิดตัวลงได้อย่างสงบเมื่อได้รับสัญญาณเช่น SIGINT (จากการกด Ctrl+C) หรือ SIGTERM แทนที่จะหยุดทำงานทันที ซึ่งอาจทำให้การเชื่อมต่อที่ยังคงอยู่ถูกตัดอย่างกะทันหัน

Tokio ให้ฟังก์ชัน `tokio::signal::ctrl_ch()` เพื่อรอสัญญาณ Ctrl+C และเราสามารถใช้มันร่วมกับ `select!` เพื่อทำการปิดการทำงานอย่างสงบได้

ตัวอย่างการปิดการทำงานอย่างสงบในเซิร์ฟเวอร์ TCP:

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{self, AsyncReadExt, AsyncWriteExt};
use tokio::signal;

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Server listening on 127.0.0.1:8080");

    // สร้าง future ที่จะเสร็จเมื่อได้รับสัญญาณ Ctrl+C
    let shutdown_signal = async {
        signal::ctrl_c().await.expect("Failed to install CTRL+C handler");
        println!("Received shutdown signal, shutting down...");
    };

    // เก็บตัวจัดการ task ทั้งหมดเพื่อที่เราจะสามารถรอให้พวกมันเสร็จสิ้นได้เมื่อปิด
    let mut tasks = Vec::new();

    loop {
        tokio::select! {
            // รอการเชื่อมต่อใหม่
            result = listener.accept() => {
                match result {
                    Ok((socket, addr)) => {
                        println!("Accepted connection from: {}", addr);
                        // สร้าง task ใหม่สำหรับการเชื่อมต่อนี้
                        let handle = tokio::spawn(process_socket(socket));
                        tasks.push(handle);
                    }
                    Err(e) => {
                        eprintln!("Failed to accept connection: {}", e);
                    }
                }
            }
            // รอสัญญาณปิดการทำงาน
            _ = shutdown_signal => {
                println!("Shutdown signal received, stopping acceptance of new connections.");
                break; // ออกจากลูปการรับการเชื่อมต่อใหม่
            }
        }
    }

    // รอให้ทุก task ที่กำลังทำงานเสร็จสิ้น
    println!("Waiting for {} active connections to finish...", tasks.len());
    for task in tasks {
        let _ = task.await;
    }

    println!("All connections closed. Goodbye!");
    Ok(())
}

async fn process_socket(mut socket: TcpStream) -> io::Result<()> {
    let mut buffer = [0; 1024];

    loop {
        let n = match socket.read(&mut buffer).await {
            Ok(0) => return Ok(()), // Client ปิดการเชื่อมต่อ
            Ok(n) => n,
            Err(e) => return Err(e),
        };

        if let Err(e) = socket.write_all(&buffer[..n]).await {
            return Err(e);
        }
    }
}
```

ในตัวอย่างข้างต้น:
1. เราสร้าง future `shutdown_signal` ที่จะเสร็จเมื่อได้รับสัญญาณ Ctrl+C
2. ในลูปหลัก เราใช้ `select!` เพื่อรอทั้งการเชื่อมต่อใหม่และสัญญาณปิดการทำงาน
3. เมื่อได้รับสัญญาณปิดการทำงาน เราออกจากลูปการรับการเชื่อมต่อใหม่ แต่ยังคงรอให้การเชื่อมต่อที่มีอยู่ทั้งหมดเสร็จสิ้นก่อนที่จะออกจากโปรแกรม
4. เราเก็บ `JoinHandle` ของทุก task ที่เราสร้างไว้ในเวกเตอร์ เพื่อที่เราจะสามารถรอให้พวกมันเสร็จสิ้นได้ด้วย `.await`

## การประยุกต์ใช้ในโปรเจกต์ wsProxy

ในโปรเจกต์ wsProxy ของเรา เราได้ใช้แนวคิดทั้งหมดที่กล่าวมาข้างต้นเพื่อสร้าง proxy ที่แปลงระหว่าง WebSocket และ TCP socket อย่างมีประสิทธิภาพ

### การตั้งค่า Tokio Runtime

ในฟังก์ชัน `main` ของ wsProxy เราได้ตั้งค่า Tokio runtime แบบ multi-threaded ด้วยจำนวน thread ที่สามารถตั้งค่าได้จากไฟล์การตั้งค่า:

```rust
fn build_runtime(threads: usize) -> tokio::runtime::Runtime {
    tokio::runtime::Builder::new_multi_thread()
        .worker_threads(threads)
        .enable_all()
        .build()
        .expect("Failed to build Tokio runtime")
}
```

จากนั้นเราใช้รันไทม์นี้เพื่อรันฟังก์ชันหลักของเรา:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = Config::load()?;
    let runtime = build_runtime(config.server.worker_threads);
    runtime.block_on(async {
        if let Err(e) = run_server(config).await {
            eprintln!("Server error: {}", e);
            std::process::exit(1);
        }
    })
}
```

### การจัดการการเชื่อมต่อด้วย `tokio::spawn` และ `tokio::select!`

เมื่อมีการเชื่อมต่อ WebSocket เข้ามา เราจะสร้าง task ใหม่เพื่อจัดการกับการเชื่อมต่อนั้นโดยเฉพาะ:

```rust
async fn handle_connection(ws_stream: WebSocketStream<MaybeTlsStream<TcpStream>>, addr: SocketAddr) {
    // แยกการเชื่อมต่อ WebSocket ออกเป็นตัวรับและตัวส่ง
    let (mut ws_sender, mut ws_receiver) = ws_stream.split();

    // เชื่อมต่อไปยัง TCP server ตามที่ตั้งค่าไว้
    let tcp_stream = TcpStream::connect(&config.server.target_addr).await?;
    let (mut tcp_reader, mut tcp_writer) = io::split(tcp_stream);

    // สร้างช่องทางสำหรับส่งสัญญาณยกเลิกระหว่างสองทิศทาง
    let (tx_to_ws, mut rx_to_ws) = mpsc::unbounded_channel();
    let (tx_to_tcp, mut rx_to_tcp) = mpsc::unbounded_channel();

    // งานที่ 1: อ่านจาก WebSocket และส่งไปยัง TCP
    let ws_to_tcp_task = tokio::spawn(async move {
        while let Some(msg) = ws_receiver.next().await {
            match msg {
                Ok(Message::Binary(data)) => {
                    if tx_to_tcp.send(data).is_err() {
                        break; // ตัวรับถูกยกเลิกแล้ว
                    }
                }
                Ok(Message::Close(_)) => {
                    // ส่งสัญญาณปิดไปยัง TCP ฝั่งผู้ส่ง
                    let _ = tx_to_tcp.send(Vec::new()).await;
                    break;
                }
                Err(e) => {
                    eprintln!("WebSocket error: {}", e);
                    break;
                }
                _ => {} // ละเลยข้อความประเภทอื่นๆ เช่น Ping, Pong, Text
            }
        }
    });

    // งานที่ 2: อ่านจาก TCP และส่งไปยัง WebSocket
    let tcp_to_ws_task = tokio::spawn(async move {
        let mut buffer = vec![0; 4096];
        loop {
            let n = match tcp_reader.read(&mut buffer).await {
                Ok(0) => break, // TCP ปิดการเชื่อมต่อ
                Ok(n) => n,
                Err(e) => {
                    eprintln!("TCP read error: {}", e);
                    break;
                }
            };

            // ส่งข้อมูลไปยัง WebSocket ผ่าน channel เพื่อหลีกเลี่ยงการล็อกบน ws_sender
            if tx_to_ws.send(buffer[..n].to_vec()).is_err() {
                break; // ตัวรับถูกยกเลิกแล้ว
            }
        }
    });

    // งานที่ 3: ส่งจาก channel ไปยัง WebSocket
    let ws_sender_task = tokio::spawn(async move {
        while let Some(data) = rx_to_ws.recv().await {
            if let Err(e) = ws_sender.send(Message::Binary(data)).await {
                eprintln!("WebSocket write error: {}", e);
                break;
            }
        }
    });

    // งานที่ 4: ส่งจาก channel ไปยัง TCP
    let tcp_sender_task = tokio::spawn(async move {
        while let Some(data) = rx_to_tcp.recv().await {
            if data.is_empty() {
                // สัญญาณปิดการเชื่อมต่อ
                break;
            }
            if let Err(e) = tcp_writer.write_all(&data).await {
                eprintln!("TCP write error: {}", e);
                break;
            }
        }
    });

    // รอให้งานทั้งสี่เสร็จสิ้น หรือเกิดข้อผิดพลาดขึ้น
    tokio::select! {
        _ = ws_to_tcp_task => {}
        _ = tcp_to_ws_task => {}
        _ = ws_sender_task => {}
        _ = tcp_sender_task => {}
    }

    // ปิดการเชื่อมต่อทั้งสองฝั่งอย่างสงบ
    let _ = ws_sender.close().await;
    let _ = tcp_writer.shutdown().await;
}
```

ในตัวอย่างข้างต้นเราจะเห็นการใช้:
1. `tokio::spawn` เพื่อสร้างงานหลายงานที่ทำงานพร้อมกัน
2. `mpsc::unbounded_channel()` เพื่อส่งข้อมูลระหว่างงานโดยไม่ต้องล็อก
3. `tokio::select!` เพื่อรอให้งานใดงานหนึ่งเสร็จสิ้นก่อน (ในกรณีนี้คือเมื่อมีงานใดงานหนึ่งพบข้อผิดพลาดหรือเสร็จสิ้น)
4. การปิดการเชื่อมต่ออย่างสงบโดยการปิดฝั่งส่งและเรียก `shutdown()` บน TCP writer

### การจัดการสัญญาณปิดการทำงาน

ในฟังก์ชัน `run_server` เราได้ตั้งค่าการฟังสัญญาณ Ctrl+C เพื่อทำการปิดการทำงานอย่างสงบ:

```rust
async fn run_server(config: Config) -> Result<(), Box<dyn std::error::Error>> {
    let addr = format!("0.0.0.0:{}", config.server.port);
    let listener = TcpListener::bind(&addr).await?;
    println!("WebSocket server listening on {}", addr);

    // สร้าง future ที่จะเสร็จเมื่อได้รับสัญญาณ Ctrl+C
    let shutdown_signal = async {
        signal::ctrl_c().await.expect("Failed to install CTRL+C handler");
        println!("\nReceived shutdown signal, shutting down gracefully...");
    };

    // เก็บตัวจัดการ task ของการเชื่อมต่อทั้งหมด
    let mut connection_tasks = Vec::new();

    // ลูปหลักสำหรับการรับการเชื่อมต่อใหม่
    loop {
        tokio::select! {
            // รอการเชื่อมต่อใหม่
            result = listener.accept() => {
                match result {
                    Ok((ws_stream, addr)) => {
                        println!("New WebSocket connection from: {}", addr);
                        // สร้าง task ใหม่สำหรับการเชื่อมต่อนี้
                        let handle = tokio::spawn(handle_connection(ws_stream, addr, config.clone()));
                        connection_tasks.push(handle);
                    }
                    Err(e) => {
                        eprintln!("Failed to accept connection: {}", e);
                    }
                }
            }
            // รอสัญญาณปิดการทำงาน
            _ = shutdown_signal => {
                println!("Shutdown signal received, stopping acceptance of new connections.");
                break; // ออกจากลูปการรับการเชื่อมต่อใหม่
            }
        }
    }

    // รอให้ทุกการเชื่อมต่อที่ยังทำงานอยู่เสร็จสิ้น
    println!("Waiting for {} active connections to close...", connection_tasks.len());
    for task in connection_tasks {
        let _ = task.await;
    }

    println!("All connections closed. Server shut down gracefully.");
    Ok(())
}
```

โครงสร้างนี้ทำให้เซิร์ฟเวอร์ของเราสามารถ:
1. ยอมรับการเชื่อมต่อใหม่ได้เรื่อยๆ จนกว่าจะได้รับสัญญาณปิดการทำงาน
2. เมื่อได้รับสัญญาณปิดการทำงาน มันจะหยุดการยอมรับการเชื่อมต่อใหม่ แต่ยังคงให้บริการการเชื่อมต่อที่มีอยู่จนกว่าจะเสร็จสิ้น
3. รอให้ทุกงานที่จัดการการเชื่อมต่อเสร็จสิ้นก่อนที่จะออกจากโปรแกรมอย่างสมบูรณ์

## สรุป

ในบทความนี้เราได้เรียนรู้เกี่ยวกับ:

1. **เหตุผลที่เราต้องการ Async programming** สำหรับ network applications โดยเฉพาะอย่างยิ่งเมื่อเทียบกับโมเดล thread-per-connection แบบดั้งเดิม
2. **พื้นฐานของ `async/await` ใน Rust** ว่ามันทำงานอย่างไรโดยการคืนค่าเป็น `Future` และการใช้ `.await` เพื่อรอผลลัพธ์
3. **Tokio runtime** วิธีการตั้งค่าโดยใช้ `#[tokio::main]` หรือ `tokio::runtime::Builder` เพื่อควบคุมจำนวน worker threads และฟีเจอร์ที่เปิดใช้งาน
4. **การสร้าง task ด้วย `tokio::spawn`** ความแตกต่างระหว่าง task และ thread และความสำคัญของการใช้ `move` closure เมื่อจำเป็น
5. **การสื่อสารระหว่าง task ด้วยช่องทาง (channels)** ทั้ง `mpsc` สำหรับการสื่อสารหลายผู้ผลิตหนึ่งผู้บริโภค และ `oneshot` สำหรับการสื่อสารหนึ่งครั้ง
6. **การทำ I/O แบบ asynchronous** ด้วย `tokio::net::TcpListener`, `TcpStream`, และ trait อย่าง `AsyncReadExt` และ `AsyncWriteExt` เพื่อสร้าง echo server แบบ async
7. **การใช้ `tokio::select!`** เพื่อรอให้หนึ่งในหลายๆ future เสร็จสิ้นก่อน ซึ่งเป็นประโยชน์อย่างยิ่งสำหรับการจัดการ timeout และการสื่อสารสองทาง
8. **การปิดการทำงานอย่างสงบ (Graceful Shutdown)** ด้วยการใช้ `tokio::signal::ctrl_c()` และการรอให้ทุกงานที่กำลังทำงานเสร็จสิ้นก่อนออกจากโปรแกรม
9. **การประยุกต์ใช้ทั้งหมดนี้ในโปรเจกต์ wsProxy** เพื่อสร้าง WebSocket-to-TCP proxy ที่มีประสิทธิภาพและสามารถจัดการการเชื่อมต่อจำนวนมากได้อย่างมีประสิทธิภาพ

แนวคิดเหล่านี้เป็นพื้นฐานสำคัญในการสร้าง network application ที่ทันสมัยและมีประสิทธิภาพสูงใน Rust ด้วยการใช้ Tokio เป็น runtime เราสามารถสร้างแอปพลิเคชันที่สามารถจัดการกับการเชื่อมต่อหลายหมื่นหรือหลักแสนการเชื่อมต่อได้โดยใช้ทรัพยากรระบบเพียงเล็กน้อยเท่านั้น

### ลิงก์ที่เกี่ยวข้อง

- ก่อนหน้า: [Traits และ Generics](/posts/rust/rust-traits-generics/)
- ถัดไป: [สร้าง wsProxy — CLI, Config และ Server](/posts/rust/rust-wsproxy-server/)