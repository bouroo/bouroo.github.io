---
title: "สร้าง wsProxy ตอนที่ 1 — CLI, Config และ Server"
subtitle: ""
date: 2026-07-13T09:00:00+07:00
lastmod: 2026-07-13T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "เริ่มสร้าง WebSocket-to-TCP Proxy ด้วย clap, axum และ tokio — CLI args, config, logging และ HTTP/WS server"
license: ""
images: []
tags: ["Rust", "Tutorial", "WebSocket", "Axum", "Tokio"]
categories: ["Rust"]
featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"
lightgallery: true
---

# สร้าง wsProxy ตอนที่ 1 — CLI, Config และ Server

สวัสดีครับ! ในบทความนี้เราจะเริ่มสร้างโปรเจกต์จริง ๆ ชื่อว่า **rs-wsProxy** ซึ่งเป็น WebSocket-to-TCP Proxy ที่ออกแบบมาสำหรับใช้งานร่วมกับ [roBrowser](https://github.com/vthibault/roBrowser) (โปรเจกต์จำลอง Ragnarok Online บนเว็บ) เพื่อให้สามารถเชื่อมต่อจากเว็บเบราว์เซอร์ไปยังเซิร์ฟเวอร์เกมได้โดยตรงผ่าน WebSocket

ย้อนกลับไปตอนที่เห็นโพสต์จาก [rayrag.com](https://rayrag.com/) บน Facebook ที่เล่น RO บนเว็บเบราว์เซอร์ได้ ผมก็เลยอยากสร้าง proxy ตัวนี้ขึ้นมาเองด้วย Rust หลังจากเรียนรู้พื้นฐานมาทั้ง 6 ตอน ถึงเวลาเอาทุกอย่างมาประกอบกันแล้ว

บทความนี้เป็น Part 1 ของซีรีส์สองส่วน โดยเราจะเน้นไปที่การตั้งค่าโครงสร้างโปรเจกต์ การตั้งค่า CLI ด้วย `clap` การจัดการการตั้งค่า (configuration) ระบบ logging ด้วย `tracing` และการตั้งค่าเซิร์ฟเวอร์ HTTP/WebSocket ด้วย `axum` และ `tokio`

ใน Part 2 (ซึ่งจะตามมาในบทความถัดไป) เราจะเจาะลึกเข้าไปในส่วนของ proxy core — กลไกการเชื่อมต่อ TCP และการส่งต่อข้อมูลระหว่าง WebSocket และ TCP socket รวมถึงการจัดการการเชื่อมต่อแบบพร้อมกันหลาย ๆ รายการ (concurrent connections) และการจัดการข้อผิดพลาด

คุณสามารถดูโค้ดต้นฉบับทั้งหมดได้ที่: https://github.com/bouroo/rs-wsProxy

<!--more-->

## โครงสร้างโปรเจกต์

มาเริ่มด้วยการสร้างโครงสร้างไฟล์ของโปรเจกต์กันก่อน โครงสร้างของ rs-wsProxy มีดังนี้:

```
rs-wsProxy/
├── src/
│   ├── main.rs       # Entry point ของแอปพลิเคชัน
│   ├── lib.rs        # ไลบรารีราก (ถ้ามี) – ในกรณีนี้เราจะใช้เป็นการประกาศโมดูล
│   ├── config.rs     # จัดการการประมวลผลอาร์กิวเมนต์บรรทัดคำสั่งและสร้างสถานะแอปพลิเคชัน (AppState)
│   ├── logging.rs    # ตั้งค่าระบบ logging ด้วย tracing
│   ├── modules.rs    # ฟังก์ชันช่วยเหลือสำหรับการตรวจสอบและแปลงเป้าหมายการเชื่อมต่อ (target validation)
│   ├── proxy.rs      # ตรรกะหลักของ proxy: การเชื่อมต่อ TCP และการส่งต่อข้อมูลระหว่าง WebSocket และ TCP
│   └── server.rs     # การตั้งค่าเซิร์ฟเวอร์ HTTP/WebSocket ด้วย axum และการกำหนดเส้นทาง (routes)
├── Cargo.toml        # Manifest ของ Cargo ที่ระบุ dependencies และ metadata
└── tests/            # โฟลเดอร์สำหรับทดสอบ (ยังว่างใน Part 1)
```

เราจะสร้างไฟล์เหล่านี้ทีละไฟล์พร้อมคำอธิบายอย่างละเอียดในภาษาไทย

## Cargo.toml — Dependencies

มาเริ่มกันที่ไฟล์ `Cargo.toml` ซึ่งเป็นไฟล์กำหนดค่าของ Cargo ที่ระบุชื่อโครงการ เวอร์ชัน เอดิชัน และที่สำคัญที่สุดคือ dependencies ที่เราจะใช้ในโปรเจกต์นี้

```toml
[package]
name = "rs-wsProxy"
version = "1.0.0"
edition = "2021"

[lib]
name = "rs_ws_proxy"
path = "src/lib.rs"

[dependencies]
axum = { version = "0.8", features = ["ws"] }
axum-server = { version = "0.8", features = ["tls-rustls"] }
tokio-rustls = "0.26"
rustls = "0.23"
tokio = { version = "1", features = ["full"] }
futures-util = "0.3"
clap = { version = "4", features = ["derive", "env"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
bytes = "1"
```

มาไล่ดูทีละตัวว่าทำไมเราถึงเลือกใช้แต่ละไลบรารี:

- **axum** (`= 0.8`, features = ["ws"]): เฟรมเวิร์กเว็บที่สร้างบนบน `tokio` และ `tower` ใช้สำหรับสร้าง HTTP server และ WebSocket endpoints ฟีเจอร์ `ws` เปิดใช้งานการสนับสนุน WebSocket
- **axum-server** (`= 0.8`, features = ["tls-rustls"]): ให้การสนับสนุน TLS ผ่าน rustls สำหรับ axum (แม้ว่าในโปรเจกต์นี้เราอาจจะไม่เปิดใช้ TLS โดยตรง แต่เราเตรียมไว้เผื่ออนาคต)
- **tokio-rustls** (`0.26`) และ **rustls** (`0.23`): ไลบรารีสำหรับการทำ TLS ทันสมัยที่ปลอดภัยและรวดเร็ว เข้ากันได้กับ tokio
- **tokio
- **tokio** (`= 1`, features = [`full`]): รันไทม์แบบอะซิงโครนัสที่ทรงพลังสำหรับการเขียนแอปพลิเคชันแบบไม่บล็อกใน Rust ฟีเจอร์ `full` จะเปิดใช้ฟีเจอร์หลักทั้งหมด (เช่น เวลา, ทาสก์, ซิงค์, ฯลฯ) ที่เราต้องการ
- **futures-util** (`0.3`): ยูทิลิตี้เสริมสำหรับทำงานกับฟิวเจอร์และสตรีม เช่น `StreamExt`, `SinkExt` ฯลฯ ซึ่งเราจะใช้ในส่วนของ proxy เพื่อทำการปั๊มข้อมูลระหว่าง WebSocket และ TCP
- **clap** (`= 4`, features = [`derive`, `env`]): ไลบรารีสำหรับสร้าง command-line interface แบบ declarative ด้วย derive macro ฟีเจอร์ `env` ทำให้เราสามารถดึงค่าพารามิเตอร์จากตัวแปรสภาพแวดล้อมได้โดยอัตโนมัติ
- **tracing** (`0.1`) และ **tracing-subscriber** (`= 0.3`, features = [`env-filter`]): ระบบ logging ที่มีโครงสร้าง (structured logging) และความสามารถในการกรองข้อความผ่าน environment variable (`RUST_LOG`) ทำให้เราสามารถควบคุมระดับการบันทึกได้อย่างละเอียดโดยไม่ต้องแก้โค้ด
- **bytes** (`1`): ไลบรารีสำหรับจัดการกับบัฟเฟอร์ไบต์ที่มีประสิทธิภาพและสามารถแชร์กันได้ (`Bytes`, `BytesMut`) ซึ่งจะมีประโยชน์มากเมื่อเราต้องทำการอ่านและเขียนข้อมูลระหว่างซ็อกเก็ต

## CLI ด้วย clap (config.rs)

ต่อไปเราจะสร้างไฟล์ `src/config.rs` ซึ่งจะประกอบด้วย:

1. โครงสร้าง `Args` ที่อนุมานจาก `clap::Parser` เพื่อรับค่าอาร์กิวเมนต์จากบรรทัดคำสั่งและตัวแปรสภาพแวดล้อม
2. โครงสร้าง `AppState` ที่จะถูกแชร์ไปทั่วทั้งแอปพลิเคชันผ่าน `std::sync::Arc`
3. ฟังก์ชันช่วยเหลือสำหรับการแปลงสตริงที่รับมาจากอาร์กิวเมนต์ (เช่น รายการเซิร์ฟเวอร์ที่อนุญาต หรือแมปของการเปลี่ยนเส้นทาง) ไปเป็นรูปแบบที่ใช้งานได้ภายในแอปพลิเคชัน

มาเริ่มด้วยการกำหนดโครงสร้าง `Args`:

```rust
use clap::Parser;
use std::collections::HashMap;
use std::net::SocketAddr;

/// WebSocket-to-TCP proxy for roBrowser
#[derive(Parser, Debug)]
#[command(name = "wsproxy", version, about = "WebSocket-to-TCP proxy")]
pub struct Args {
    /// พอร์ตที่เซิร์ฟเวอร์จะฟังอยู่ (ค่าเริ่มต้น: 5999)
    #[arg(short = 'p', long = "port", env = "WSPROXY_PORT", default_value_t = 5999)]
    pub port: u16,

    /// จำนวนเวิร์กเกอร์ทาสก์สำหรับ tokio runtime (ค่าเริ่มต้น: 1)
    #[arg(short = 't', long = "threads", env = "WSPROXY_THREADS", default_value_t = 1)]
    pub threads: usize,

    /// เปิดใช้งาน SSL/TLS (ต้องมีใบรับรองและคีย์)
    #[arg(short = 's', long = "ssl", env = "WSPROXY_SSL")]
    pub ssl: bool,

    /// รายการเซิร์ฟเวอร์ที่อนุญาตให้เชื่อมต่อได้ (คั่นด้วยเครื่องหมาย comma) ตัวอย่าง: "game1.example.com:80,game2.example.com:443"
    /// หากไม่ระบุ (None) จะอนุญาตให้เชื่อมต่อได้ทุกเซิร์ฟเวอร์ (open proxy)
    /// หากระบุเป็นสตริงว่าง ("") หรือเวกเตอร์ว่าง จะไม่อนุญาตให้เชื่อมต่อใด ๆ เลย
    #[arg(short = 'a', long = "allow", env = "WSPROXY_ALLOW")]
    pub allow: Option<String>,

    /// กฎการเปลี่ยนเส้นทาง (redirect rules) ในรูปแบบ "from=to;from2=to2"
    /// ตัวอย่าง: "game.example.com:80=game-secure.example.com:443"
    #[arg(short = 'r', long = "redirect", env = "WSPROXY_REDIRECT")]
    pub redirect: Option<String>,

    /// เซิร์ฟเวอร์เป้าหมายเริ่มต้นเมื่อไม่มีการระบุเป้าหมายใน URL (เช่น เมื่อเชื่อมต่อที่ /ws แทนที่จะเป็น /ws/game.example.com:80)
    #[arg(short = 'd', long = "default-target", env = "WSPROXY_DEFAULT_TARGET")]
    pub default_target: Option<String>,

    /// ที่อยู่ที่จะบินด์เซิร์ฟเวอร์ (ค่าเริ่มต้น: 0.0.0.0 หมายถึงฟังทุกอินเทอร์เฟซ)
    #[arg(long = "host", env = "WSPROXY_HOST", default_value = "0.0.0.0")]
    pub host: String,
}
```

### คำอธิบายโครงสร้าง `Args`

- เราใช้ `derive(Parser)` จากไลบรารี `clap` เพื่อให้มันสร้างโค้ดสำหรับการวิเคราะห์อาร์กิวเมนต์บรรทัดคำสั่งให้เราโดยอัตโนมัติ จากฟิลด์และแอททริบิวต์ที่เรากำหนด
- แต่ละฟิลด์มีแอททริบิวต์ `arg` ที่กำหนดชื่อสั้น (`short`), ชื่อยาว (`long`), ตัวแปรสภาพแวดล้อม (`env`), และค่าเริ่มต้น (`default_value_t` หรือ `default_value`)
- ฟิลด์ `port`, `threads` มีค่าเริ่มต้นเป็นตัวเลข (`default_value_t`)
- ฟิลด์ `ssl` เป็นบูลีนแบบสวิตช์ (flag) – หากมีการระบุ flag จะเป็น `true` ไม่เช่นนั้นจะเป็น `false`
- ฟิลด์ `allow`, `redirect`, `default_target` เป็น `Option<String>` เพื่อให้สามารถไม่ระบุค่าได้ (คือ `None`)
- ฟิลด์ `host` มีค่าเริ่มต้นเป็น `"0.0.0.0"` ซึ่งหมายถึงการฟังบนทุกอินเทอร์เฟซเครือข่ายที่มีอยู่

ต่อไปเราจะกำหนดโครงสร้าง `AppState` ซึ่งจะถูกแชร์ไปทั่วทั้งแอปพลิเคชันโดยใช้ `std::sync::Arc` เพื่อให้สามารถเข้าถึงได้อย่างปลอดภัยจากหลายทาสก์ (เนื่องจากเราใช้ `tokio` ซึ่งทำงานแบบอะซิงโครนัสและอาจมีหลายงานที่ทำงานพร้อมกัน)

```rust
use std::collections::HashMap;

/// สถานะที่แชร์กันทั่วทั้งแอปพลิเคชัน
#[derive(Debug, Clone)]
pub struct AppState {
    /// รายการเซิร์ฟเวอร์ที่อนุญาตให้เชื่อมต่อได้ (ถ้าเป็น None หมายถึงอนุญาตทุกเซิร์ฟเวอร์)
    /// ถ้าเป็น Some(Vec) แต่เวกเตอร์ว่าง จะหมายถึงไม่อนุญาตเซิร์ฟเวอร์ใด ๆ เลย
    pub allowed_servers: Option<Vec<String>>,

    /// แมปของกฎการเปลี่ยนเส้นทาง: จากรูปแบบต้นทางไปยังรูปแบบปลายทาง
    /// ตัวอย่าง: "game.example.com:80" -> "game-secure.example.com:443"
    pub redirects: HashMap<String, String>,

    /// เซิร์ฟเวอร์เป้าหมายเริ่มต้นเมื่อไม่มีการระบุเป้าหมายใน URL
    pub default_target: Option<String>,
}
```

#### คำอธิบายฟิลด์ใน `AppState`

- `allowed_servers`: เป็น `Option<Vec<String>>` เพื่อให้เราสามารถแสดงสามสถานะได้อย่างชัดเจน:
  - `None` (ไม่ได้ตั้งค่า) → เปิด proxy (อนุญาตให้เชื่อมต่อเซิร์ฟเวอร์ใด ๆ ก็ได้)
  - `Some(vec![])` (เวกเตอร์ว่าง) → ปฏิเสธการเชื่อมต่อทั้งหมด (ไม่มีเซิร์ฟเวอร์ใดได้รับอนุญาต)
  - `Some(vec![s1, s2, ...])` → อนุญาตเฉพาะเซิร์ฟเวอร์ที่อยู่ในรายการนี้เท่านั้น
- `redirects`: ใช้ `HashMap<String, String>` เพื่อจับคู่ระหว่างเป้าหมายต้นทางกับเป้าหมายปลายทาง ตัวอย่างเช่น อาจใช้เพื่อบังคับให้การเชื่อมต่อไปยังเซิร์ฟเวอร์เก่าเป็นการเชื่อมต่อไปยังเซิร์ฟเวอร์ใหม่ที่มีการเข้ารหัส
- `default_target`: ใช้เมื่อผู้ใช้เชื่อมต่อมาที่เส้นทางเช่น `/ws` (โดยไม่ระบุเป้าหมาย) แทนที่จะเป็น `/ws/game.example.com:80` ในกรณีนี้เราจะใช้ค่าที่ตั้งค่านี้เป็นเซิร์ฟเวอร์เป้าหมายเริ่มต้น

ต่อไปเราจะเพิ่มฟังก์ชันช่วยเหลือสำหรับการแปลงสตริงที่รับมาจากอาร์กิวเมนต์บรรทัดคำสั่งให้อยู่ในรูปแบบที่เหมาะสมกับ `AppState`

```rust
use std::net::SocketAddr;

impl Args {
    /// แปลงสตริงที่คั่นด้วยเครื่องหมาย comma ที่ได้จาก `--allow` ให้เป็น `Option<Vec<String>>`
    /// หากสตริงเป็น None หรือว่างเปล่า จะคืนค่า None
    /// หากสตริงไม่ว่าง จะแยกด้วย comma, trim แต่ละรายการ และกรองรายการว่างออก
    pub fn build_allowed_list(&self) -> Option<Vec<String>> {
        self.allow.as_ref().filter(|s| !s.is_empty()).map(|s| {
            s.split(',')
                .map(|s| s.trim())
                .filter(|s| !s.is_empty())
                .map(String::from)
                .collect()
        })
    }

    /// แปลงสตริงที่ได้จาก `--redirect` ให้เป็น `HashMap<String, String>`
    /// รูปแบบที่คาดหวัง: "from1=to1;from2=to2"
    /// หากสตริงเป็น None หรือว่างเปล่า จะคืนค่า HashMap ว่าง
    pub fn build_redirects(&self) -> HashMap<String, String> {
        self.redirect
            .as_ref()
            .map(|s| {
                s.split(';')
                    .filter(|pair| !pair.is_empty())
                    .filter_map(|pair| {
                        let mut parts = pair.splitn(2, '=');
                        let from = parts.next().map(|s| s.trim());
                        let to = parts.next().map(|s| s.trim());
                        match (from, to) {
                            (Some(from), Some(to)) if !from.is_empty() && !to.is_empty() => {
                                Some((from.to_string(), to.to_string()))
                            }
                            _ => None,
                        }
                    })
                    .collect()
            })
            .unwrap_or_default()
    }

    /// แปลงสตริงโฮสต์และพอร์ต (เช่น "game.example.com:80") ให้เป็น `SocketAddr`
    /// คืนค่า `Ok(SocketAddr)` หากสำเร็จ หรือ `Err(String)` หากล้มเหลว
    pub fn parse_socket_addr(&self, addr: &str) -> Result<SocketAddr, String> {
        addr.parse::<SocketAddr>()
            .map_err(|e| format!("ไม่สามารถแยกวิเคราะห์ที่อยู่ '{}' เป็น SocketAddr ได้: {}", addr, e))
    }

    /// สร้างอินสแตนซ์ของ `AppState` จากอาร์กิวเมนต์ที่ได้รับ
    pub fn into_app_state(self) -> AppState {
        AppState {
            allowed_servers: self.build_allowed_list(),
            redirects: self.build_redirects(),
            default_target: self.default_target.filter(|s| !s.is_empty()),
        }
    }
}
```

### คำอธิบายฟังก์ชันช่วยเหลือใน `impl Args`

- `build_allowed_list`: 
  - ตรวจสอบว่าฟิลด์ `allow` เป็น `Some` และไม่ใช่สตริงว่าง หากไม่เงื่อนไขนี้ให้คืนค่า `None`
  - หากมีค่า เราจะแยกสตริงด้วยเครื่องหมาย comma (`,`), ตัดช่องว่างด้านหน้าและหลังแต่ละส่วน (`trim`), กรองออกรายการที่ว่างเปล่าหลังการตัดช่องว่าง, จากนั้นแปลงแต่ละรายการเป็น `String` และเก็บลงในเวกเตอร์
  - ผลลัพธ์คือ `Option<Vec<String>>` ที่สอดคล้องกับนิยามของฟิลด์ `allowed_servers` ใน `AppState`

- `build_redirects`:
  - หากฟิลด์ `redirect` เป็น `None` หรือว่างเปล่า ให้คืนค่า `HashMap` ว่าง
  - หากมีค่า เราจะแยกสตริงด้วยเครื่องหมายจุดไข่ปลา (`;`) เพื่อได้คู่ `from=to` หลาย ๆ คู่
  - สำหรับแต่ละคู่ เราจะแยกอีกครั้งด้วยเครื่องหมายเท่ากับ (`=`) โดยใช้ `splitn(2, '=')` เพื่อให้ได้สูงสุดสองส่วน (ป้องกันกรณีที่มี `=` มากกว่าหนึ่งตัวในค่า `to`)
  - ตรวจสอบว่าทั้ง `from` และ `to` มีอยู่และไม่ว่างหลังจากการตัดช่องว่าง แล้วจึงแปลงเป็น `String` และใส่ลงใน `HashMap`
  - ส่งคืน `HashMap` ที่สร้างขึ้น

- `parse_socket_addr`:
  - ฟังก์ชันยูทิลิตี้ง่าย ๆ ที่พยายามแยกวิเคราะห์สตริงที่อยู่ (เช่น `"example.com:80"`) ให้เป็น `std::net::SocketAddr`
  - หากล้มเหลวจะคืนค่า `Err` พร้อมข้อความแสดงข้อผิดพลาดที่อธิบายได้ชัดเจน
  - ฟังก์ชันนี้จะถูกใช้ในภายหลังเมื่อเราต้องตรวจสอบว่าผู้ใช้ป้อนที่อยู่ในรูปแบบที่ถูกต้องหรือไม่ (เช่น ในเส้นทาง WebSocket)

- `into_app_state`:
  - แปลงอินสแตนซ์ของ `Args` ที่ได้จากการประมวลผลอาร์กิวเมนต์บรรทัดคำสั่งให้กลายเป็น `AppState` ที่พร้อมใช้งานทั่วทั้งแอปพลิเคชัน
  - เรียกใช้ฟังก์ชันช่วยเหลือสองฟังก์ชันด้านบนเพื่อสร้าง `allowed_servers` และ `redirects`
  - สำหรับ `default_target` เราจะใช้ `filter` เพื่อแปลงสตริงว่างให้เป็น `None` (เนื่องจากใน `AppState` เราต้องการให้เป็น `Option<String>` และต้องการปฏิบัติต่อสตริงว่างเหมือนไม่ได้ตั้งค่า)

ตอนนี้เรามีส่วนจัดการการตั้งค่าและอาร์กิวเมนต์บรรทัดคำสั่งแล้ว ต่อไปเราจะสร้างไฟล์ `src/logging.rs` เพื่อตั้งค่าระบบ logging ด้วย `tracing` และ `tracing-subscriber`

## Logging (logging.rs)

การทำ logging ที่ดีเป็นสิ่งสำคัญสำหรับการดีบักและการตรวจสอบแอปพลิเคชันเซิร์ฟเวอร์ ในไฟล์นี้เราจะตั้งค่า `tracing` subscriber ที่สามารถอ่านระดับการบันทึกจากตัวแปรสภาพแวดล้อม `RUST_LOG` ได้ (ตามมาตรฐานของ `tracing`)

```rust
use tracing_subscriber::{fmt, EnvFilter};

/// ตั้งค่าระบบ logging ด้วย tracing
/// 
/// ฟังก์ชันนี้ควรถูกเรียกใช้ครั้งเดียวที่จุดเริ่มต้นของแอปพลิเคชัน (ใน main)
/// มันจะตั้งค่า subscriber ของ tracing เพื่อส่งเหตุการณ์ไปยังมาตรฐานเอาต์พุต (stdout)
/// โดยใช้รูปแบบที่อ่านง่าย และอนุญาตให้กรองระดับการบันทึกผ่านตัวแปรสภาพแวดล้อม RUST_LOG
///
/// ตัวอย่างการใช้งาน:
/// ```bash
/// RUST_LOG=info,wsproxy=debug ./wsproxy
/// ```
/// จะแสดงข้อความในระดับ info ขึ้นไปทั่วไป และระดับ debug สำหรับโมดูลที่ชื่อขึ้นต้นด้วย `wsproxy`
pub fn init() {
    // สร้างตัวกรองจากตัวแปรสภาพแวดล้อม RUST_LOG (หากไม่ได้ตั้งค่า จะใช้ "info" เป็นค่าเริ่มต้น)
    let filter = EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| EnvFilter::new("info"));

    // ตั้งค่า subscriber ของ tracing
    tracing_subscriber::fmt()
        .with_env_filter(filter)
        .init();
}
```

### คำอธิบายฟังก์ชัน `init`

- เราใช้ `EnvFilter::try_from_default_env()` เพื่อพยายามอ่านตัวแปรสภาพแวดล้อม `RUST_LOG` หากไม่พบหรือไม่สามารถแยกวิเคราะห์ได้ เราจะใช้ตัวกรองเริ่มต้นที่ระดับ `"info"`
- จากนั้นเราสร้าง `tracing_subscriber::fmt()` ซึ่งเป็นผู้ส่งออกเหตุการณ์ไปยัง stdout ในรูปแบบที่อ่านง่าย (มีสี หากเทอร์มินัลรองรับ)
- เราใช้ `.with_env_filter(filter)` เพื่อนำตัวกรองที่เราสร้างมาใช้
- สุดท้ายเราเรียก `.init()` เพื่อตั้งค่า subscriber นี้ให้เป็นตัวจัดการเหตุการณ์เริ่มต้นของ `tracing`

เมื่อมีการตั้งค่าแล้ว เราสามารถใช้มาโคร `tracing::info!`, `tracing::warn!`, `tracing::error!`, `tracing::debug!` และอื่น ๆ ในโค้ดของเราเพื่อบันทึกเหตุการณ์ต่าง ๆ ได้อย่างง่ายดาย

ต่อไปเราจะสร้างไฟล์ `src/modules.rs` ซึ่งจะมีฟังก์ชันช่วยเหลือสำหรับการตรวจสอบและแปลงเป้าหมายการเชื่อมต่อ (target validation) ซึ่งเป็นส่วนสำคัญของการทำหน้าที่เป็น proxy อย่างปลอดภัย

## Verify Pipeline (modules.rs)

ในส่วนนี้เราจะกำหนดฟังก์ชันสองฟังก์ชันหลัก:

1. `validate_target` – ตรวจสอบว่าสตริงที่อยู่ (เช่น `"example.com:80"`) มีรูปแบบที่ถูกต้องเป็น `SocketAddr` หรือไม่
2. `verify` – ตรรกะหลักสำหรับการตรวจสอบว่าไคลเอนต์ได้รับอนุญาตให้เชื่อมต่อไปยังเป้าหมายที่ร้องขอหรือไม่ โดยพิจารณาจาก:
   - กฎการเปลี่ยนเส้นทาง (redirect rules) – หากตรงกับกฎใดกฎหนึ่ง จะใช้เป้าหมายที่ถูกแทนที่ตามกฎนั้น
   - รายการเซิร์ฟเวอร์ที่อนุญาต (allow list) – หากมีการตั้งค่ารายการนี้ จะต้องตรวจสอบว่าเป้าหมาย (หลังจากการเปลี่ยนเส้นทางแล้ว) อยู่ในรายการนี้
   - หากไม่มีการตั้งค่ารายการอนุญาต (คือ `None`) จะถือว่าเป็น open proxy (อนุญาตให้เชื่อมต่อได้ทุกที่)

```rust
use std::net::SocketAddr;
use crate::config::AppState;

/// ตรวจสอบว่าสตริงที่อยู่มีรูปแบบที่ถูกต้องเป็น SocketAddr หรือไม่
/// 
/// ฟังก์ชันนี้จะพยายามแยกวิเคราะห์สตริงที่ให้มาเป็น SocketAddr
/// หากสำเร็จจะคืนค่า Ok(SocketAddr) หากล้มเหลวจะคืนค่า Err พร้อมข้อความอธิบาย
pub fn validate_target(target: &str) -> Result<SocketAddr, String> {
    target
        .parse::<SocketAddr>()
        .map_err(|e| format!("ที่อยู่ '{}' ไม่ถูกต้อง: {}", target, e))
}

/// ตรวจสอบและอาจปรับเปลี่ยนเป้าหมายการเชื่อมต่อตามกฎที่ตั้งค่าไว้ใน AppState
/// 
/// ขั้นตอนการทำงาน:
/// 1. ตรวจสอบก่อนว่าสตริงเป้าหมายเดิมมีรูปแบบที่ถูกต้อง (โดยใช้ validate_target)
/// 2. ตรวจสอบกฎการเปลี่ยนเส้นทาง (redirects) ใน AppState:
///    - หากพบกฎที่ตรงกับเป้าหมายต้นทาง (จาก) ให้แทนที่เป้าหมายด้วยปลายทาง (ไป) ตามกฎนั้น
///    - หากพบหลายกฎที่ตรงกัน จะใช้กฎแรกที่พบ (เนื่องจากเราใช้ HashMap ซึ่งไม่รับประกันลำดับ แต่ในทางปฏิบัติเราควรออกแบบกฎให้ไม่ซ้อนทับกัน)
/// 3. ตรวจสอบรายการเซิร์ฟเวอร์ที่อนุญาต (allowed_servers) ใน AppState:
///    - หากเป็น None → อนุญาตให้เชื่อมต่อได้ทุกที่ (open proxy)
///    - หากเป็น Some(vec) แต่เวกเตอร์ว่าง → ไม่อนุญาตให้เชื่อมต่อที่ไหนเลย
///    - หากเป็น Some(vec) ที่มีรายการ → ตรวจสอบว่าเป้าหมาย (หลังจากการเปลี่ยนเส้นทางแล้ว) อยู่ในรายการนี้หรือไม่
/// 
/// พารามิเตอร์:
///   - `state`: สถานะแอปพลิเคชันที่มีกฎการตั้งค่าต่าง ๆ
///   - `target`: สตริงที่อยู่ต้นทางที่ไคลเอนต์ต้องการเชื่อมต่อไป (เช่น "game.example.com:80")
/// 
/// ส่งคืน:
///   - Ok(SocketAddr) หากการเชื่อมต่อได้รับอนุญาต (อาจมีการเปลี่ยนเส้นทางแล้ว)
///   - Err(String) หากการเชื่อมต่อถูกปฏิเสธ พร้อมข้อความอธิบายเหตุผล
pub fn verify(state: &AppState, target: &str) -> Result<SocketAddr, String> {
    // ขั้นตอนที่ 1: ตรวจสอบรูปแบบของที่อยู่ต้นทาง
    let mut addr = validate_target(target)?;

    // ขั้นตอนที่ 2: ตรวจสอบกฎการเปลี่ยนเส้นทาง (ถ้ามี)
    if let Some(from) = state.redirects.get(target) {
        // พบกฎการเปลี่ยนเส้นทางที่ตรงกันทั้งสตริง
        addr = validate_target(from)?;
    } else {
        // หากไม่พบการตรงกันแบบเต็มสตริง เราอาจต้องการตรวจสอบแบบขึ้นต้นด้วยหรือไม่?
        // ในการออกแบบปัจจุบันเราใช้การตรงกันแบบเต็มสตริงเท่านั้น
        // หากต้องการสนับสนุนการจับคู่แบบขึ้นต้นด้วย เราจำเป็นต้องเปลี่ยนการตรวจสอบนี้
        // แต่สำหรับตอนนี้เราจะใช้การจับคู่แบบเต็มสตริงเท่านั้น
    }

    // ขั้นตอนที่ 3: ตรวจสอบรายการเซิร์ฟเวอร์ที่อนุญาต (ถ้ามีการตั้งค่า)
    match &state.allowed_servers {
        None => {
            // ไม่มีการตั้งค่ารายการอนุญาต → อนุญาตให้เชื่อมต่อได้ทุกที่ (open proxy)
            Ok(addr)
        }
        Some(list) => {
            if list.is_empty() {
                // รายการอนุญาตว่างเปล่า → ไม่อนุญาตให้เชื่อมต่อที่ไหนเลย
                Err(format!(
                    "การเชื่อมต่อไปยัง {} ถูกปฏิเสธ: ไม่มีเซิร์ฟเวอร์ใดได้รับอนุญาต (allow list ว่างเปล่า)",
                    target
                ))
            } else {
                // ตรวจสอบว่าที่อยู่ (หลังจากการเปลี่ยนเส้นทางแล้ว) อยู่ในรายการอนุญาตหรือไม่
                // เนื่องจากเราเก็บที่อยู่ในรูปแบบ String ใน allowed_servers เราจึงต้องแปลง addr กลับเป็น String เพื่อเปรียบเทียบ
                let addr_str = addr.to_string();
                if list.contains(&addr_str) {
                    Ok(addr)
                } else {
                    Err(format!(
                        "การเชื่อมต่อไปยัง {} ถูกปฏิเสธ: ไม่อยู่ในรายการเซิร์ฟเวอร์ที่อนุญาต",
                        target
                    ))
                }
            }
        }
    }
}
```

### คำอธิบายฟังก์ชันใน `modules.rs`

- `validate_target`:
  - เป็นฟังก์ชันยูทิลิตี้ที่เรียกใช้ `SocketAddr::parse` จากไลบรารีมาตรฐานของ Rust
  - หากการแยกวิเคราะห์สำเร็จ จะคืนค่า `Ok(SocketAddr)`
  - หากล้มเหลว จะคืนค่า `Err` พร้อมข้อความอธิบายที่ช่วยให้ผู้ใช้เข้าใจว่าทำไมที่อยู่ที่ให้มาจึงไม่ถูกต้อง (เช่น ขาดพอร์ต, มีอักขระที่ไม่ได้รับอนุญาต ฯลฯ)

- `verify`:
  - นี่คือฟังก์ชันหลักที่ใช้ในการตัดสินใจว่าจะอนุญาตให้เชื่อมต่อไปยังเป้าหมายที่ร้องขอหรือไม่
  - ขั้นตอนการทำงานถูกแบ่งออกเป็นสามขั้นตอนหลักตามที่อธิบายไว้ในคอมเมนต์
  - ขั้นตอนที่ 1: ตรวจสอบรูปแบบของที่อยู่ต้นทางโดยใช้ `validate_target`
  - ขั้นตอนที่ 2: ตรวจสอบกฎการเปลี่ยนเส้นทาง (redirects)
    - เราใช้ `HashMap::get` เพื่อค้นหาเป้าหมายต้นทางที่แน่นอนในแมปของกฎการเปลี่ยนเส้นทาง
    - หากพบ เราจะแทนที่ที่อยู่ต้นทางด้วยที่อยู่ปลายทางจากกฎนั้น และทำการตรวจสอบรูปแบบใหม่อีกครั้ง (เผื่อว่ากฎการเปลี่ยนเส้นทางอาจให้ค่าที่ไม่ถูกต้อง)
    - หมายเหตุ: ในการออกแบบปัจจุบันเราใช้การจับคู่แบบเต็มสตริงเท่านั้น หากต้องการสนับสนุนการจับคู่แบบขึ้นต้นด้วยหรือแบบนิพจน์ปกติ เราจำเป็นต้องปรับเปลี่ยนตรงนี้ แต่สำหรับการใช้งานทั่วไปของ roBrowser การจับคู่แบบเต็มสตริงก็เพียงพอแล้ว
  - ขั้นตอนที่ 3: ตรวจสอบรายการเซิร์ฟเวอร์ที่อนุญาต (allowed_servers)
    - หาก `allowed_servers` เป็น `None` หมายถึงไม่มีการตั้งค่ารายการอนุญาต → ถือเป็น open proxy (อนุญาตให้เชื่อมต่อได้ทุกที่)
    - หากเป็น `Some(vec)` แต่เวกเตอร์ว่างเปล่า → หมายถึงผู้ใช้ต้องการปฏิเสธการเชื่อมต่อทั้งหมด (อาจใช้สำหรับการปิดชั่วคราวหรือการตั้งค่าที่ผิดโดยไม่ตั้งใจ)
    - หากเป็น `Some(vec)` ที่มีองค์ประกอบหนึ่งขึ้นไป → เราจะตรวจสอบว่าที่อยู่ปลายทาง (หลังจากการเปลี่ยนเส้นทางแล้ว) อยู่ในรายการนี้หรือไม่
      - เพื่อทำการเปรียบเทียบเราจำเป็นต้องแปลง `SocketAddr` กลับเป็นสตริงโดยใช้ `to_string()` เนื่องจากเราเก็บรายการอนุญาตเป็นเวกเตอร์ของสตริง
      - หากพบในรายการ → คืนค่า `Ok(addr)` (อนุญาตให้เชื่อมต่อ)
      - หากไม่พบในรายการ → คืนค่า `Err` พร้อมข้อความอธิบายว่าการเชื่อมต่อถูกปฏิเสธเนื่องจากไม่อยู่ในรายการที่อนุญาต

ต่อไปเราจะสร้างไฟล์ `src/server.rs` ซึ่งจะประกอบด้วยการตั้งค่าเซิร์ฟเวอร์ HTTP/WebSocket ด้วยเฟรมเวิร์ก `axum` รวมถึงการกำหนดเส้นทาง (routes) และผู้จัดการเหตุการณ์ (handlers)

## Axum Server (server.rs)

ในไฟล์นี้เราจะกำหนด:

1. เส้นทาง (routes) สำหรับเซิร์ฟเวอร์ HTTP:
   - `GET /` – เส้นทางหลักที่คืนค่าข้อความต้อนรับสั้น ๆ
   - `GET /ws` – เส้นทาง WebSocket สำหรับการเชื่อมต่อไปยังเซิร์ฟเวอร์เป้าหมายเริ่มต้น (หากมีการตั้งค่า)
   - `GET /{*target}` – เส้นทาง WebSocket ที่รับพาธแบบไวด์การ์ดเพื่อระบุเซิร์ฟเวอร์เป้าหมายแบบไดนามิก (เช่น `/ws/game.example.com:80`)

2. ผู้จัดการเหตุการณ์ (handlers) สำหรับแต่ละเส้นทาง:
   - `get_root` – ส่งข้อความต้อนรับกลับเป็นข้อความธรรมดา
   - `ws_upgrade_default` – จัดการการอัปเกรดเป็น WebSocket สำหรับเส้นทาง `/ws` โดยใช้เซิร์ฟเวอร์เป้าหมายเริ่มต้นจากสถานะแอปพลิเคชัน
   - `ws_upgrade` – จัดการการอัปเกรดเป็น WebSocket สำหรับเส้นทางแบบไวด์การ์ด โดยดึงเป้าหมายจากพาธและตรวจสอบผ่านฟังก์ชัน `verify` จาก `modules.rs`

3. ฟังก์ชัน `create_app` ที่ประกอบทุกอย่างเข้าด้วยกันโดยสร้าง `axum::Router` และเพิ่มสถานะแอปพลิเคชันเข้าไปด้วย `with_state`

```rust
use std::net::SocketAddr;
use axum::{
    extract::{Path, State},
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use axum::extract::ws::{Message, WebSocket, WebSocketUpgrade};
use futures_util::{SinkExt, StreamExt};
use tracing::{info, warn};

use crate::config::AppState;
use crate::modules::verify;

/// เส้นทางหลักของเซิร์ฟเวอร์ – คืนค่าข้อความต้อนรับสั้น ๆ
async fn get_root() -> &'static str {
    "Welcome to rs-wsProxy! WebSocket-to-TCP proxy for roBrowser."
}

/// จัดการการอัปเกรดเป็น WebSocket สำหรับเส้นทาง `/ws` (ใช้เซิร์ฟเวอร์เป้าหมายเริ่มต้น)
async fn ws_upgrade_default(
    State(state): State<AppState>,
    ws: WebSocketUpgrade,
) -> impl IntoResponse {
    // หากมีการตั้งค่าเซิร์ฟเวอร์เป้าหมายเริ่มต้น ให้ใช้มัน ไม่เช่นนั้นคืนค่าข้อผิดพลาด
    let target = match &state.default_target {
        Some(t) if !t.is_empty() => t.as_str(),
        _ => {
            return Err("ไม่มีการตั้งค่าเซิร์ฟเวอร์เป้าหมายเริ่มต้น".to_string());
        }
    };

    // ตรวจสอบว่าการเชื่อมต่อไปยังเป้าหมายเริ่มต้นนี้ได้รับอนุญาตหรือไม่
    match verify(state, target) {
        Ok(addr) => {
            // หากได้รับอนุญาต ให้ดำเนินการอัปเกรดเป็น WebSocket และส่งที่อยู่ที่ตรวจสอบแล้วไปยังตัวจัดการ WebSocket
            Ok(ws.on_upgrade(move |socket| handle_socket(socket, addr)))
        }
        Err(e) => Err(format!("การเชื่อมต่อถูกปฏิเสธ: {}", e)),
    }
}

/// จัดการการอัปเกรดเป็น WebSocket สำหรับเส้นทางแบบไวด์การ์ด `/ {*target}`
async fn ws_upgrade(
    State(state): State<AppState>,
    Path(target): Path<String>,
    ws: WebSocketUpgrade,
) -> impl IntoResponse {
    // ตรวจสอบว่าการเชื่อมต่อไปยังเป้าหมายที่ระบุในพาธนี้ได้รับอนุญาตหรือไม่
    match verify(state, &target) {
        Ok(addr) => {
            // หากได้รับอนุญาต ให้ดำเนินการอัปเกรดเป็น WebSocket และส่งที่อยู่ที่ตรวจสอบแล้วไปยังตัวจัดการ WebSocket
            Ok(ws.on_upgrade(move |socket| handle_socket(socket, addr)))
        }
        Err(e) => Err(format!("การเชื่อมต่อถูกปฏิเสธ: {}", e)),
    }
}

/// จัดการการสื่อสารผ่าน WebSocket หลังจากการอัปเกรดสำเร็จ
/// 
/// ฟังก์ชันนี้จะรับ `WebSocket` stream และ `SocketAddr` ของเซิร์ฟเวอร์เป้าหมาย
/// จากนั้นจะสร้างการเชื่อมต่อ TCP ไปยังที่อยู่นั้น และทำการปั๊มข้อมูลสองทางระหว่าง WebSocket และ TCP socket
async fn handle_socket(mut ws: WebSocket, addr: SocketAddr) {
    // บันทึกข้อมูลการเชื่อมต่อใหม่
    info!("รับการเชื่อมต่อ WebSocket ใหม่ ไปยัง {}", addr);

    // สร้างการเชื่อมต่อ TCP ไปยังเซิร์ฟเวอร์เป้าหมาย
    match tokio::net::TcpStream::connect(addr).await {
        Ok(mut tcp_stream) => {
            info!("เชื่อมต่อ TCP ไปยัง {} สำเร็จ", addr);

            // แยก WebSocket ออกเป็นสตรีมผู้ส่ง (sink) และผู้รับ (stream)
            let (mut ws_sender, mut ws_receiver) = ws.split();

            // แยก TcpStream ออกเป็นผู้อ่านและผู้เขียน
            let (mut tcp_reader, mut tcp_writer) = tokio::io::split(tcp_stream);

            // งานที่ 1: ส่งข้อมูลจาก WebSocket ไปยัง TCP socket
            let ws_to_tcp = async {
                while let Some(msg) = ws_receiver.next().await {
                    match msg {
                        Ok(Message::Binary(data)) => {
                            // หากได้รับข้อมูลไบนารีจาก WebSocket ให้เขียนลง TCP socket
                            if let Err(e) = tcp_writer.write_all(&data).await {
                                warn!("เกิดข้อผิดพลาดในการเขียนข้อมูลไปยัง TCP socket: {}", e);
                                break;
                            }
                        }
                        Ok(Message::Close(_)) => {
                            // หากได้รับสัญญาณปิดการเชื่อมจาก WebSocket ให้หยุดลูป
                            break;
                        }
                        Err(e) => {
                            // หากเกิดข้อผิดพลาดในการรับข้อมูลจาก WebSocket
                            warn!("เกิดข้อผิดพลาดในการรับข้อมูลจาก WebSocket: {}", e);
                            break;
                        }
                        _ => {
                            // ละเลยข้อความประเภทอื่น ๆ (เช่น Text, Ping, Pong) ในตัวอย่างนี้
                            // ในการใช้งานจริงอาจต้องจัดการกับข้อความประเภทอื่น ๆ ตามความเหมาะสม
                        }
                    }
                }
                // เมื่อลูปสิ้นสุด ให้พยายามปิดการเขียนไปยัง TCP socket
                let _ = tcp_writer.shutdown().await;
            };

            // งานที่ 2: ส่งข้อมูลจาก TCP socket ไปยัง WebSocket
            let tcp_to_ws = async {
                let mut buffer = vec![0u8; 4096]; // บัฟเฟอร์ชั่วคราวสำหรับอ่านข้อมูลจาก TCP socket
                loop {
                    match tcp_reader.read(&mut buffer).await {
                        Ok(0) => {
                            // อ่านได้ 0 ไบต์ หมายถึงการเชื่อมต่อ TCP ถูกปิดโดยเซิร์ฟเวอร์ปลายทาง
                            break;
                        }
                        Ok(n) => {
                            // ได้รับข้อมูลจาก TCP socket ให้ส่งเป็นข้อมูลไบนารีไปยัง WebSocket
                            if ws_sender
                                .send(Message::Binary(binary::Bytes::copy_from_slice(&buffer[..n]))).await
                                .is_err()
                            {
                                // หากการส่งล้มเหลว (อาจเป็นเพราะ WebSocket ถูกปิดแล้ว) ให้หยุดลูป
                                break;
                            }
                        }
                        Err(e) => {
                            // เกิดข้อผิดพลาดในการอ่านจาก TCP socket
                            warn!("เกิดข้อผิดพลาดในการอ่านข้อมูลจาก TCP socket: {}", e);
                            break;
                        }
                    }
                }
                // เมื่อลูปสิ้นสุด ให้พยายามปิด WebSocket connection
                let _ = ws_sender.send(Message::Close(None)).await;
            };

            // รันงานทั้งสองพร้อมกัน และรอให้ทั้งสองงานเสร็จสิ้น (หรือใดงานหนึ่งล้มเหลว)
            tokio::select! {
                _ = ws_to_tcp => {},
                _ = tcp_to_ws => {},
            }

            info!("การเชื่อมต่อกับ {} ถูกปิดแล้ว", addr);
        }
        Err(e) => {
            // หากไม่สามารถเชื่อมต่อ TCP ไปยังเซิร์ฟเวอร์เป้าหมายได้
            warn!("ไม่สามารถเชื่อมต่อไปยัง {}: {}", addr, e);
            // แจ้งให้ไคลเอนต์ทราบว่าการเชื่อมต่อล้มเหลวโดยการปิด WebSocket ด้วยรหัสสถานะภายใน
            let _ = ws.send(Message::Close(None)).await;
        }
    }
}

/// สร้างแอปพลิเคชัน Axum พร้อมเส้นทางและสถานะที่แชร์กัน
pub fn create_app(state: AppState) -> Router {
    Router::new()
        // เส้นทางหลักสำหรับตรวจสอบว่าเซิร์ฟเวอร์ทำงานอยู่
        .route("/", get(get_root))
        // เส้นทาง WebSocket สำหรับเชื่อมต่อไปยังเซิร์ฟเวอร์เป้าหมายเริ่มต้น (ถ้ามีการตั้งค่า)
        .route("/ws", get(ws_upgrade_default))
        // เส้นทาง WebSocket แบบไวด์การ์ดเพื่อระบุเซิร์ฟเวอร์เป้าหมายแบบไดนามิกจากพาธ
        // ตัวอย่าง: /ws/game.example.com:80
        .route("/ws/*target", get(ws_upgrade))
        // เพิ่มสถานะแอปพลิเคชันเข้าไปในเราเตอร์ เพื่อให้ผู้จัดการเหตุการณ์สามารถเข้าถึงได้ผ่านการสกัด State
        .with_state(state)
}
```

### คำอธิบายโค้ดใน `server.rs`

- การนำเข้า (imports):
  - เรานำเข้าฟังก์ชันและโครงสร้างที่จำเป็นจาก `axum` สำหรับการสร้างเราเตอร์ การกำหนดเส้นทาง และการสกัดข้อมูลจากคำขอ (เช่น `State`, `Path`)
  - เรานำเข้า `WebSocket`, `WebSocketUpgrade` และประเภทข้อความ WebSocket จาก `axum::extract::ws`
  - เรานำเข้า `SinkExt` และ `StreamExt` จาก `futures_util` เพื่อให้สามารถใช้เมธอดเช่น `.next()` บนสตรีมและ `.send()` บนซิงค์ได้อย่างสะดวก
  - เรานำเข้า `tracing` มาใช้สำหรับการบันทึกเหตุการณ์ต่าง ๆ ระหว่างการทำงานของเซิร์ฟเวอร์
  - เรานำเข้าโครงสร้าง `AppState` จากโมดูล `config` และฟังก์ชัน `verify` จากโมดูล `modules`

- ผู้จัดการเหตุการณ์ (handlers):
  - `get_root`: เพียงแค่คืนค่าสตริงต้อนรับเป็นการตอบสนอง HTTP ธรรมดา
  - `ws_upgrade_default`: จัดการการร้องขอ WebSocket ไปยังเส้นทาง `/ws`
    - ดึงสถานะแอปพลิเคชันจากตัวสกัด `State`
    - ตรวจสอบว่ามีการตั้งค่า `default_target` ในสถานะหรือไม่ หากมีและไม่ว่างเปล่า ให้ใช้ค่านั้นเป็นเป้าหมาย
    - หากไม่มีการตั้งค่า `default_target` ให้คืนค่าข้อผิดพลาด
    - ตรวจสอบว่าได้รับอนุญาตให้เชื่อมต่อไปยังเป้าหมายนั้นหรือไม่โดยใช้ฟังก์ชัน `verify` จากโมดูล `modules`
    - หากได้รับอนุญาต ให้เรียกใช้ `ws.on_upgrade` และส่งการเชื่อมต่อ WebSocket ที่อัปเกรดแล้วและที่อยู่ที่ตรวจสอบแล้วไปยังฟังก์ชัน `handle_socket`
    - หากไม่ได้รับอนุญาต ให้คืนค่าข้อผิดพลาดพร้อมข้อความอธิบาย
  - `ws_upgrade`: คล้ายกับ `ws_upgrade_default` แต่รับเป้าหมายจากพาธของ URL โดยใช้ตัวสกัด `Path`
    - ตัวอย่าง: หากผู้ใช้เชื่อมต่อมาที่ `/ws/game.example.com:80` ตัวสกัด `Path` จะดึงสตริง `"game.example.com:80"` ไปยังตัวแปร `target`
    - จากนั้นทำการตรวจสอบเหมือนกับใน `ws_upgrade_default`

- `handle_socket`: ฟังก์ชันที่จัดการการสื่อสารจริง ๆ หลังจากที่ WebSocket ถูกอัปเกรดสำเร็จแล้ว
  - รับพารามิเตอร์สองอย่าง: `ws` ซึ่ง是 `WebSocket` stream ที่เชื่อมต่อกับไคลเอนต์ และ `addr` ซึ่ง是 `SocketAddr` ของเซิร์ฟเวอร์เป้าหมายที่ได้รับการตรวจสอบแล้ว
  - บันทึกข้อมูลเมื่อมีการเชื่อมต่อใหม่เข้ามาโดยใช้ `tracing::info!`
  - พยายามสร้างการเชื่อมต่อ TCP ไปยังเซิร์ฟเวอร์เป้าหมายโดยใช้ `tokio::net::TcpStream::connect`
    - หากสำเร็จ ให้ดำเนินการต่อไปยังการตั้งค่าการส่งผ่านข้อมูลสองทาง
    - หากล้มเหลว ให้บันทึกคำเตือนและพยายามปิดการเชื่อมต่อ WebSocket โดยส่งข้อความ Close
  - เมื่อเชื่อมต่อ TCP สำเร็จ เราจะแบ่งการทำงานออกเป็นสองงานหลักที่ทำงานพร้อมกันโดยใช้ `tokio::select!`:
    1. **WebSocket → TCP (`ws_to_tcp`)**:
       - แยก WebSocket ออกเป็นผู้ส่ง (`ws_sender`) และผู้รับ (`ws_receiver`) โดยใช้เมธอด `split()`
       - วนลูปเพื่อรับข้อความจาก WebSocket ผ่าน `ws_receiver.next()`
       - หากได้รับข้อความไบนารี (`Message::Binary`) ให้เขียนข้อมูลนั้นลงใน TCP socket ผ่าน `tcp_writer.write_all()`
       - หากได้รับข้อความปิดการเชื่อม (`Message::Close`) ให้ออกจากลูป
       - หากเกิดข้อผิดพลาดในการรับข้อความจาก WebSocket ให้บันทึกคำเตือนและออกจากลูป
       - หลังออกจากลูป ให้พยายามปิดการเขียนไปยัง TCP socket โดยเรียก `tcp_writer.shutdown()`
    2. **TCP → WebSocket (`tcp_to_ws`)**:
       - แยก TcpStream ออกเป็นผู้อ่าน (`tcp_reader`) และผู้เขียน (`tcp_writer`) โดยใช้ `tokio::io::split`
       - สร้างบัฟเฟอร์ชั่วคราวขนาด 4096 ไบต์สำหรับอ่านข้อมูลจาก TCP socket
       - วนลูปเพื่ออ่านข้อมูลจาก TCP socket เข้าสู่บัฟเฟอร์โดยใช้ `tcp_reader.read()`
       - หากอ่านได้ 0 ไบต์ หมายถึงการเชื่อมต่อ TCP ถูกปิดโดยเซิร์ฟเวอร์ปลายทาง ให้ออกจากลูป
       - หากอ่านได้ข้อมูลจำนวน `n` ไบต์ ให้พยายามส่งเป็นข้อความไบนารี (`Message::Binary`) ไปยัง WebSocket ผ่าน `ws_sender.send()`
       - หากการส่งล้มเหลว (อาจเป็นเพราะ WebSocket ถูกปิดแล้ว) ให้ออกจากลูป
       - หากเกิดข้อผิดพลาดในการอ่านจาก TCP socket ให้บันทึกคำเตือนและออกจากลูป
       - หลังออกจากลูป ให้พยายามส่งข้อความปิดการเชื่อมไปยัง WebSocket โดยเรียก `ws_sender.send(Message::Close(None))`

  - งานทั้งสองนี้จะทำงานพร้อมกันโดยใช้ `tokio::select!` ซึ่งจะรอจนกว่างานใดงานหนึ่งจะเสร็จสิ้น (ไม่ว่าจะเสร็จสมบูรณ์หรือเนื่องจากข้อผิดพลาด) จากนั้นจึงออกจากฟังก์ชัน
  - เมื่อทั้งสองงานเสร็จสิ้น ให้บันทึกข้อมูลว่าการเชื่อมต่อกับเซิร์ฟเวอร์เป้าหมายถูกปิดแล้ว

- `create_app`: ฟังก์ชันที่สร้างและคืนค่า `axum::Router` ที่พร้อมใช้งาน
  - สร้างเราเตอร์ใหม่โดยใช้ `Router::new()`
  - เพิ่มเส้นทาง GET สำหรับเส้นทางหลัก (`"/"`) ที่เชื่อมโยงกับผู้จัดการเหตุการณ์ `get_root`
  - เพิ่มเส้นทาง GET สำหรับเส้นทาง WebSocket เริ่มต้น (`"/ws"`) ที่เชื่อมโยงกับผู้จัดการเหตุการณ์ `ws_upgrade_default`
  - เพิ่มเส้นทาง GET สำหรับเส้นทาง WebSocket แบบไวด์การ์ด (`"/ws/*target"`) ที่เชื่อมโยงกับผู้จัดการเหตุการณ์ `ws_upgrade`
    - เครื่องหมาย `*` ในพาธหมายถึง "จับคู่ทุกอย่างที่เหลืออยู่ในพาธและส่งเป็นพารามิเตอร์"
    - ในกรณีนี้ พาธ `/ws/game.example.com:80` จะทำให้ตัวแปร `target` ในผู้จัดการเหตุการณ์ได้รับค่า `"game.example.com:80"`
  - สุดท้าย เราใช้เมธอด `with_state` เพื่อแนบสถานะแอปพลิเคชัน (`state`) เข้ากับเราเตอร์ เพื่อให้ผู้จัดการเหตุการณ์ทุกตัวสามารถเข้าถึงสถานะนี้ได้ผ่านการสกัด `State<AppState>`

ต่อไปเราจะสร้างไฟล์หลักของแอปพลิเคชัน `src/main.rs` ซึ่งจะเป็นจุดเริ่มต้นของโปรแกรม ที่นี่เราจะ:

1. เริ่มต้นระบบ logging
2. ประมวลผลอาร์กิวเมนต์บรรทัดคำสั่งโดยใช้ `clap`
3. ตั้งค่า runtime ของ `tokio` ด้วยจำนวนเวิร์กเกอร์ที่กำหนด
4. สร้างสถานะแอปพลิเคชันจากอาร์กิวเมนต์
5. สร้างแอปพลิเคชัน Axum โดยใช้สถานะนั้น
6. ผูกเซิร์ฟเวอร์กับที่อยู่และพอร์ตที่ระบุ
7. จัดการสัญญาณเพื่อปิดเซิร์ฟเวอร์อย่างนุ่มนวลเมื่อได้รับ SIGINT หรือ SIGTERM

## main.rs — Entry Point

```rust
use std::net::SocketAddr;
use std::sync::Arc;
use tokio::signal;
use tokio::sync::Notify;
use tracing::info;

use clap::Parser;
use tokio::net::TcpListener;

use rs_ws_proxy::config::Args;
use rs_ws_proxy::logging;
use rs_ws_proxy::server::create_app;

/// ฟังก์ชันหลักของแอปพลิเคชัน
#[tokio::main]
async fn main() {
    // 1. เริ่มต้นระบบ logging
    logging::init();
    info!("เริ่มต้น rs-wsProxy...");

    // 2. ประมวลผลอาร์กิวเมนต์บรรทัดคำสั่ง
    let args = Args::parse();
    info!(
        "กำลังเริ่มเซิร์ฟเวอร์บน {}:{}, threads = {}, ssl = {}",
        args.host, args.port, args.threads, args.ssl
    );

    // บันทึกการตั้งค่าต่าง ๆ หากมีการระบุ
    if let Some(ref allow) = args.allow {
        info!("อนุญาตให้เชื่อมต่อเฉพาะ: {}", allow);
    } else {
        info!("โหมดเปิด proxy (อนุญาตให้เชื่อมต่อได้ทุกที่)");
    }

    if let Some(ref redirect) = args.redirect {
        if !redirect.is_empty() {
            info!("กฎการเปลี่ยนเส้นทาง: {}", redirect);
        }
    }

    if let Some(ref default_target) = args.default_target {
        if !default_target.is_empty() {
            info!("เซิร์ฟเวอร์เป้าหมายเริ่มต้น: {}", default_target);
        }
    }

    // 3. สร้างสถานะแอปพลิเคชันจากอาร์กิวเมนต์
    let state = Arc::new(args.into_app_state());

    // 4. สร้างแอปพลิเคชัน Axum
    let app = create_app(Arc::clone(&state));

    // 5. สร้างที่อยู่ที่จะบินด์เซิร์ฟเวอร์
    let addr = SocketAddr::new(
        args.host
            .parse()
            .expect("ไม่สามารถแยกวิเคราะห์ที่อยู่อินเทอร์เฟซได้"),
        args.port,
    );

    // 6. สร้าง TCP listener
    let listener = TcpListener::bind(&addr)
        .await
        .expect(&format!("ไม่สามารถผูกเซิร์ฟเวอร์กับที่อยู่ {}", addr));

    info!("เซิร์ฟเวอร์กำลังฟังที่ {}", addr);

    // 7. สร้างสัญญาณแจ้งเตือนสำหรับการปิดเซิร์ฟเวอร์อย่างนุ่มนวล
    let shutdown_signal = Arc::new(Notify::new());
    let shutdown_signal_clone = shutdown_signal.clone();

    // เริ่มต้นงานที่รอสัญญาณการปิดเซิร์ฟเวอร์ (SIGINT หรือ SIGTERM)
    let shutdown_task = tokio::spawn(async move {
        // รอสัญญาณจากระบบปฏิบัติการ
        tokio::select! {
            _ = signal::ctrl_c() => {
                info!("ได้รับสัญญาณ SIGINT (Ctrl+C), กำลังปิดเซิร์ฟเวอร์...");
            }
            _ = signal::unix::signal(tokio::signal::unix::SignalKind::terminate()) => {
                info!("ได้รับสัญญาณ SIGTERM, กำลังปิดเซิร์ฟเวอร์...");
            }
        }
        // แจ้งให้งานหลักทราบว่าถึงเวลาปิดเซิร์ฟเวอร์แล้ว
        shutdown_signal_clone.notify_one();
    });

    // 8. เริ่มต้นเซิร์ฟเวอร์โดยใช้ hyper ผ่าน axum
    // เราใช้ `axum::serve` ซึ่งจะรับการเชื่อมต่อ TCP จาก listener และจัดการด้วยแอปพลิเคชัน Axum ของเรา
    let server_task = tokio::spawn(async move {
        // เรียกใช้ axum::serve เพื่อเริ่มรับการเชื่อมต่อ
        if let Err(e) = axum::serve(listener, app.into_make_service()).await {
            eprintln!("เซิร์ฟเวอร์ทำงานผิดพลาด: {}", e);
        }
    });

    // 9. รอให้ทั้งงานเซิร์ฟเวอร์และงานรอสัญญาณปิดเซิร์ฟเวอร์เสร็จสิ้น
    tokio::select! {
        _ = server_task => {
            info!("งานเซิร์ฟเวอร์สิ้นสุดลง");
        }
        _ = async {
            shutdown_signal.notified().await;
        } => {
            // เมื่อได้รับสัญญาณให้ปิดเซิร์ฟเวอร์ เราจะทำการปิด listener อย่างชัดเจน
            // ซึ่งจะทำให้ axum หยุดรับการเชื่อมต่อใหม่และรอให้การเชื่อมต่อที่มีอยู่เสร็จสิ้น
            drop(listener);
        }
    }

    // รองานที่เหลือให้เสร็จสิ้นก่อนออกจากโปรแกรม
    let _ = shutdown_task.await;
    let _ = server_task.await;

    info!("เซิร์ฟเวอร์ถูกปิดลงอย่างปลอดภัยแล้ว");
}
```

### คำอธิบายโค้ดใน `main.rs`

- การนำเข้า (imports):
  - เรานำเข้าฟังก์ชันและโครงสร้างที่จำเป็นจากไลบรารีมาตรฐานของ Rust เช่น `std::net::SocketAddr` สำหรับที่อยู่เครือข่าย และ `std::sync::Arc` สำหรับการนับจำนวนการอ้างอิงแบบอะตอมิกเพื่อให้สามารถแชร์สถานะระหว่างทาสก์ได้อย่างปลอดภัย
  - เรานำเข้า `tokio::signal` สำหรับการรอสัญญาณจากระบบปฏิบัติการ (เช่น SIGINT จากการกด Ctrl+C) และ `tokio::sync::Notify` สำหรับการส่งสัญญาณระหว่างงานภายในโปรแกรมของเราเอง
  - เรานำเข้า `tracing::info` เพื่อใช้ในการบันทึกเหตุการณ์สำคัญต่าง ๆ ระหว่างการทำงานของเซิร์ฟเวอร์
  - เรานำเข้าฟังก์ชันและโครงสร้างจากไลบรารีภายนอกที่เราได้เพิ่มเข้าไปใน `Cargo.toml`:
    - `clap::Parser` สำหรับการประมวลผลอาร์กิวเมนต์บรรทัดคำสั่ง
    - `tokio::net::TcpListener` สำหรับการสร้าง TCP listener ที่จะรอรับการเชื่อมต่อเข้ามา
    - โมดูลของเราเอง: `rs_ws_proxy::config::Args` (โครงสร้างอาร์กิวเมนต์), `rs_ws_proxy::logging` (ฟังก์ชันเริ่มต้น logging), และ `rs_ws_proxy::server::create_app` (ฟังก์ชันสร้างแอปพลิเคชัน Axum)

- ฟังก์ชันหลัก `main`:
  - เราทำเครื่องหมายฟังก์ชันนี้ด้วยแอททริบิวต์ `#[tokio::main]` เพื่อให้โทคิโอตั้งค่ารันไทม์แบบอัตโนมัติเมื่อโปรแกรมเริ่มทำงาน
  - ขั้นตอนที่ 1: เริ่มต้นระบบ logging โดยเรียกใช้ `logging::init()` ที่เราได้กำหนดไว้ใน `src/logging.rs` จากนั้นบันทึกข้อความว่าเริ่มต้นโปรแกรมแล้ว
  - ขั้นตอนที่ 2: ประมวลผลอาร์กิวเมนต์บรรทัดคำสั่งโดยเรียกใช้ `Args::parse()` จากไลบรารี `clap` ซึ่งจะทำการวิเคราะห์อาร์กิวเมนต์จากบรรทัดคำสั่งและตัวแปรสภาพแวดล้อมตามที่เราได้กำหนดไว้ในโครงสร้าง `Args` จากนั้นบันทึกข้อมูลการตั้งค่าต่าง ๆ เช่น ที่อยู่และพอร์ตที่จะบินด์ จำนวนเวิร์กเกอร์ สถานะ SSL และการตั้งค่าต่าง ๆ ที่เกี่ยวกับการอนุญาต การเปลี่ยนเส้นทาง และเซิร์ฟเวอร์เป้าหมายเริ่มต้น (หากมีการระบุ)
  - ขั้นตอนที่ 3: สร้างสถานะแอปพลิเคชันโดยเรียกใช้เมธอด `into_app_state()` บนออบเจกต์ `args` ที่ได้จากการ parse แล้ว แปลงผลลัพธ์ให้เป็น `Arc<AppState>` เพื่อให้สามารถแชร์สถานะนี้ไปยังทาสก์ต่าง ๆ ได้อย่างปลอดภัยโดยใช้การอ้างอิงแบบนับจำนวน
  - ขั้นตอนที่ 4: สร้างแอปพลิเคชัน Axum โดยเรียกใช้ฟังก์ชัน `create_app` ที่เราได้กำหนดไว้ใน `src/server.rs` และส่งสถานะแอปพลิเคชันที่เราได้สร้างไว้ให้กับมัน ผลลัพธ์ที่ได้คือ `axum::Router` ที่พร้อมใช้งาน
  - ขั้นตอนที่ 5: สร้างที่อยู่ที่จะบินด์เซิร์ฟเวอร์โดยแยกวิเคราะห์สตริงที่อยู่จากอาร์กิวเมนต์ `args.host` เป็น `std::net::IpAddr` และรวมกับพอร์ต `args.port` เพื่อสร้าง `SocketAddr` หากการแยกวิเคราะห์ที่อยู่ล้มเหลว เราจะใช้ `expect()` เพื่อทำให้โปรแกรมหยุดทำงานและแสดงข้อความข้อผิดพลาด
  - ขั้นตอนที่ 6: สร้าง TCP listener โดยใช้ `tokio::net::TcpListener::bind` กับที่อยู่ที่เราได้สร้างไว้ จากนั้นรอให้การผูกที่อยู่สำเร็จโดยใช้ `.await` หากล้มเหลวเราจะใช้ `expect()` เพื่อหยุดโปรแกรมและแสดงข้อความที่ระบุว่าทำไม่สามารถผูกเซิร์ฟเวอร์กับที่อยู่นั้นได้
  - ขั้นตอนที่ 7: เตรียมการจัดการสัญญาณเพื่อปิดเซิร์ฟเวอร์อย่างนุ่มนวล
    - เราสร้าง `Arc<Notify::new()>` ซึ่งเป็นสัญญาณแจ้งเตือนที่สามารถแชร์ระหว่างงานได้อย่างปลอดภัย
    - เราโคลนสัญญาณนี้เพื่อให้งานหลักและงานรอสัญญาณสามารถใช้ร่วมกันได้
    - เราสร้างงานใหม่โดยใช้ `tokio::spawn` ที่จะรอให้เกิดสัญญาณจากระบบปฏิบัติการไม่ว่าจะเป็น SIGINT (จากการกด Ctrl+C) หรือ SIGTERM (สัญญาณสิ้นสุดการทำงาน) เมื่อได้รับสัญญาณใดสัญญาณหนึ่ง เราจะบันทึกข้อความและส่งสัญญาณแจ้งเตือนไปยัง `shutdown_signal` เพื่อบอกให้งานหลักทราบว่าถึงเวลาปิดเซิร์ฟเวอร์แล้ว
  - ขั้นตอนที่ 8: เริ่มต้นเซิร์ฟเวอร์โดยใช้ `axum::serve`
    - เราสร้างงานใหม่โดยใช้ `tokio::spawn` อีกครั้งเพื่อรันเซิร์ฟเวอร์
    - ภายในงานนี้ เราเรียกใช้ `axum::serve(listener, app.into_make_service())` ซึ่งจะทำให้เซิร์ฟเวอร์เริ่มรับการเชื่อมต่อ TCP จาก `listener` ที่เราได้สร้างไว้ และจัดการแต่ละการเชื่อมต่อโดยใช้แอปพลิเคชัน Axum ที่เราได้สร้างไว้ (แปลงเป็น `MakeService` โดยใช้ `.into_make_service()`)
    - หากเซิร์ฟเวอร์พบข้อผิดพลาดขณะทำงาน เราจะพิมพ์ข้อความข้อผิดพลาดออกไปยังมาตรฐานข้อผิดพลาด (stderr)
  - ขั้นตอนที่ 9: รอให้ทั้งงานเซิร์ฟเวอร์และงานรอสัญญาณปิดเซิร์ฟเวอร์เสร็จสิ้น
    - เราใช้ `tokio::select!` เพื่อรอให้งานใดงานหนึ่งจากสองงานนี้เสร็จสิ้นก่อน:
      - หากงานเซิร์ฟเวอร์เสร็จสิ้นก่อน (ซึ่งอาจเกิดขึ้นได้หากมีข้อผิดพลาดร้ายแรง) เราจะบันทึกข้อความว่างานเซิร์ฟเวอร์สิ้นสุดลง
      - หากงานรอสัญญาณเสร็จสิ้นก่อน (ซึ่งหมายความว่าเราได้รับสัญญาณให้ปิดเซิร์ฟเวอร์) เราจะทำการปิด `listener` อย่างชัดเจนโดยใช้ `drop(listener)` ซึ่งจะทำให้ `axum` หยุดรับการเชื่อมต่อใหม่และรอให้การเชื่อมต่อที่มีอยู่ทั้งหมดเสร็จสิ้นก่อนที่จะปิดตัวเอง
    - หลังจากที่ `tokio::select! เสร็จสิ้น เราจะรองานที่เหลือให้เสร็จสิ้นโดยใช้ `let _ = shutdown_task.await;` และ `let _ = server_task.await;` เพื่อให้แน่ใจว่าทุกอย่างถูกทำความสะอาดอย่างเหมาะสมก่อนที่โปรแกรมจะออก
    - สุดท้ายเราบันทึกข้อความว่าเซิร์ฟเวอร์ถูกปิดลงอย่างปลอดภัยแล้ว

## ทดสอบการทำงาน

ตอนนี้เรามีโครงสร้างพื้นฐานของโปรเจกต์ rs-wsProxy แล้ว ลองมาทดสอบรันโปรแกรมกันดู

ก่อนอื่น ให้แน่ใจว่าคุณอยู่ในไดเรกทอรีรากของโปรเจกต์ แล้วรันคำสั่งต่อไปนี้เพื่อคอมไพล์และรันโปรแกรมด้วยการตั้งค่าเริ่มต้น:

```bash
cargo run
```

คุณควรเห็นผลลัพธ์ประมาณนี้:

```
[INFO  wsproxy] เริ่มต้น rs-wsProxy...
[INFO  wsproxy] กำลังเริ่มเซิร์ฟเวอร์บน 0.0.0.0:5999, threads = 1, ssl = false
[INFO  wsproxy] โหมดเปิด proxy (อนุญาตให้เชื่อมต่อได้ทุกที่)
[INFO  wsproxy] เซิร์ฟเวอร์กำลังฟังที่ 0.0.0.0:5999
```

จากนั้นในเทอร์มินัลอีกหน้าต่างหนึ่ง คุณสามารถทดสอบโดยใช้ `curl` เพื่อเรียกใช้เส้นทาง HTTP ปกติ:

```bash
curl http://localhost:5999/
```

คุณควรได้รับคำตอบว่า:

```
Welcome to rs-wsProxy! WebSocket-to-TCP proxy for roBrowser.
```

ต่อมาคุณสามารถทดสอบการเชื่อมต่อ WebSocket โดยใช้เครื่องมืออย่าง `websocat` (ถ้ายังไม่ได้ติดตั้ง ให้ติดตั้งโดยใช้ `cargo install websocat` หรือใช้แพ็กเกจจ์ของระบบปฏิบัติการของคุณ):

```bash
websocat ws://localhost:5999/ws
```

เนื่องจากเรายังไม่ได้ตั้งค่าเซิร์ฟเวอร์เป้าหมายเริ่มต้น (`default_target`) คุณจะเห็นข้อผิดพลาดในการเชื่อมต่อ เนื่องจากเซิร์ฟเวอร์จะพยายามตรวจสอบว่าการเชื่อมต่อไปยังเซิร์ฟเวอร์เป้าหมายเริ่มต้นได้รับอนุญาตหรือไม่ แต่เนื่องจากไม่มีการตั้งค่าเอาไว้ มันจึงคืนค่าข้อผิดพลาด

ลองตั้งค่าเซิร์ฟเวอร์เป้าหมายเริ่มต้นโดยใช้ตัวแปรสภาพแวดล้อมหรืออาร์กิวเมนต์บรรทัดคำสั่งดู:

```bash
WSPROXY_DEFAULT_TARGET="echo.websocket.org:443" cargo run
```

จากนั้นทดสอบการเชื่อมต่อ WebSocket อีกครั้ง:

```bash
websocat ws://localhost:5999/ws
```

คราวนี้คุณควรเห็นการเชื่อมต่อสำเร็จไปยัง `echo.websocket.org:443` (ซึ่งเป็นเซิร์ฟเวอร์ WebSocket สาธิตที่ส่งกลับข้อความที่คุณส่งไป) และคุณสามารถทดสอบโดยพิมพ์ข้อความใด ๆ ลงไปในเทอร์มินัลของ `websocat` แล้วมันจะส่งกลับมาทันที

คุณยังสามารถทดสอบการระบุเซิร์ฟเวอร์เป้าหมายแบบไดนามิกผ่านพาธได้ด้วย เช่น หากคุณต้องการเชื่อมต่อไปยัง `echo.websocket.org:443` โดยตรงโดยไม่ต้องพึ่งพาการตั้งค่าเริ่มต้น:

```bash
websocat ws://localhost:5999/ws/echo.websocket.org:443
```

หากคุณต้องการทดสอบฟีเจอร์การอนุญาตเฉพาะเซิร์ฟเวอร์บางตัว คุณสามารถรันเซิร์ฟเวอร์ด้วยการตั้งค่า `allow` ดังนี้:

```bash
WSPROXY_ALLOW="echo.websocket.org:443" WSPROXY_DEFAULT_TOKEN="echo.websocket.org:443" cargo run
```

จากนั้นทดสอบการเชื่อมต่อไปยังเซิร์ฟเวอร์ที่ได้รับอนุญาต:

```bash
websocat ws://localhost:5999/ws/echo.websocket.org:443
```

ควรเชื่อมต่อสำเร็จ

และทดสอบการเชื่อมต่อไปยังเซิร์ฟเวอร์ที่ไม่อนุญาต:

```bash
ws://localhost:5999/ws/google.com:443
```

ควรถูกปฏิเสธโดยเซิร์ฟเวอร์ และคุณจะเห็นข้อความแสดงข้อผิดพลาดในไคลเอนต์ websocat

## สรุป

ในบทความนี้เราได้สร้างพื้นฐานของโปรเจกต์ rs-wsProxy ขึ้นมาแล้ว โดยเราได้:

1. สร้างโครงสร้างไฟล์ของโปรเจกต์อย่างเป็นระบบ
2. กำหนด dependencies ที่จำเป็นใน `Cargo.toml` และอธิบายเหตุผลว่าเราเลือกใช้ไลบรารีแต่ละตัวเพราะเหตุผลใด
3. สร้างระบบจัดการอาร์กิวเมนต์บรรทัดคำสั่งและสภาพแวดล้อมด้วย `clap` ในไฟล์ `src/config.rs` รวมถึงการกำหนดโครงสร้าง `Args` และ `AppState` พร้อมฟังก์ชันช่วยเหลือสำหรับการแปลงและตรวจสอบค่าต่าง ๆ
4. ตั้งค่าระบบ logging ด้วย `tracing` และ `tracing-subscriber` ในไฟล์ `src/logging.rs`
5. สร้างฟังก์ชันช่วยเหลือสำหรับการตรวจสอบและยืนยันความถูกต้องของเป้าหมายการเชื่อมต่อในไฟล์ `src/modules.rs` รวมถึงตรรกะสำหรับการเปลี่ยนเส้นทางและการตรวจสอบรายการอนุญาต
6. สร้างเซิร์ฟเวอร์ HTTP/WebSocket ด้วย `axum` และ `tokio` ในไฟล์ `src/server.rs` โดยกำหนดเส้นทางสำหรับหน้าหลัก เว็บซ็อกเก็ตเริ่มต้น และเว็บซ็อกเก็ตแบบไดนามิกผ่านพาธ
7. เขียนฟังก์ชันหลักใน `src/main.rs` ที่ทำหน้าที่เริ่มต้นระบบ logging ประมวลผลอาร์กิวเมนต์ สร้างสถานะแอปพลิเคชัน สร้างแอปพลิเคชัน Axum ผูกเซิร์ฟเวอร์กับที่อยู่และพอร์ต และจัดการการปิดเซิร์ฟเวอร์อย่างนุ่มนวลเมื่อได้รับสัญญาณจากระบบปฏิบัติการ

เราได้ทดสอบการทำงานเบื้องต้นโดยการรันเซิร์ฟเวอร์และทดสอบการเชื่อมต่อด้วย `curl` และ `websocat` ซึ่งแสดงให้เห็นว่าระบบพื้นฐานของเราทำงานได้ตามที่คาดหวัง

ในบทความถัดไป (Part 2) เราจะเจาะลึกเข้าไปในส่วนของ proxy core — กลไกการเชื่อมต่อ TCP และการส่งต่อข้อมูลระหว่าง WebSocket และ TCP socket ซึ่งเราได้วางโครงสร้างคร่ ๆ ไว้ในฟังก์ชัน `handle_socket` ใน `src/server.rs` แต่เราจะปรับปรุงให้มีความแข็งแรงมากขึ้น จัดการกับข้อผิดพลาดได้ดียิ่งขึ้น และเพิ่มฟีเจอร์ต่าง ๆ เช่น การจัดการความดันย้อนกลับ (backpressure) และการหมดเวลาเชื่อมต่อ (timeouts)

อย่าลืมติดตามตอนต่อไปนะครับ!

### ลิงก์ที่เกี่ยวข้อง

- บทความก่อนหน้า: [Async Rust กับ Tokio](/posts/rust/rust-async-tokio/)
- บทความถัดไป: [สร้าง wsProxy — Proxy Core และ Deployment](/posts/rust/rust-wsproxy-proxy-deploy/)
- ซอร์สโค้ด: https://github.com/bouroo/rs-wsProxy