---
title: "สร้าง wsProxy ตอนที่ 2 — Proxy Core และ Deployment"
subtitle: ""
date: 2026-07-14T09:00:00+07:00
lastmod: 2026-07-14T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: " ส่วนประกอบสุดท้ายของ wsProxy: TCP connection, bidirectional pump, TLS, testing, Docker และ Kubernetes deployment"
license: ""
images: []
tags: ["Rust", "Tutorial", "WebSocket", "Tokio", "Docker"]
categories: ["Rust"]
featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"
lightgallery: true
---

# สร้าง wsProxy ตอนที่ 2 — Proxy Core และ Deployment

บทสุดท้าย! จากโพสต์ [rayrag.com](https://rayrag.com/) ที่เห็นบน Facebook จนอยากสร้าง WebSocket-to-TCP proxy ของตัวเอง เราได้เรียนรู้ Rust มาทั้ง 7 ตอน และสร้าง CLI, config, server ไปแล้ว ตอนนี้เราจะสร้างหัวใจของ proxy — TCP connection และ bidirectional pump ที่ส่งข้อมูลระหว่าง WebSocket และ TCP แล้วปิดด้วย testing และ deployment ให้พร้อมใช้งานจริง

<!--more-->

ซอร์สโค้ด: https://github.com/bouroo/rs-wsProxy

## TCP Connection (proxy.rs)

ในส่วนนี้เราจะดูการเชื่อมต่อ TCP ไปยังเซิร์ฟเวอร์เป้าหมาย ฟังก์ชัน `connect_tcp` มีหน้าที่ทำการ resolve ชื่อโดเมนเป็นที่อยู่ IP และสร้างการเชื่อมต่อ TCP โดยเปิดใช้งาน TCP_NODELAY เพื่อลดความหน่วงในการส่งข้อมูลเล็กน้อย

```rust
pub async fn connect_tcp(addr: &str) -> Result<TcpStream, String> {
    // ทำการ resolve ชื่อโดเมนเป็นรายการที่อยู่ IP
    let addrs: Vec<_> = tokio::net::lookup_host(addr)
        .await
        .map_err(|e| format!("DNS lookup failed for '{}': {}", addr, e))?
        // กรองเฉพที่อยู่ IPv4 เท่านั้น (เราไม่รองรับ IPv6 ในตัวอย่างนี้)
        .filter(|a| a.is_ipv4())
        .collect();

    // ตรวจสอบว่ามีที่อยู่ที่เหลือหลังจากกรองหรือไม่
    if addrs.is_empty() {
        return Err(format!("No IPv4 addresses found for {}", addr));
    }

    // พยายามเชื่อมต่อไปยังแต่ละที่อยู่จนกว่าจะสำเร็จ
    for addr in addrs {
        match TcpStream::connect(&addr).await {
            Ok(stream) => {
                // เปิดใช้งาน TCP_NODELAY เพื่อปิด Nagle's algorithm
                // ทำให้ข้อมูลเล็กๆ ถูกส่งทันทีโดยไม่รอให้เต็ม buffer
                if let Err(e) = stream.set_nodelay(true) {
                    return Err(format!("Failed to set TCP_NODELAY: {}", e));
                }
                return Ok(stream);
            }
            Err(e) => {
                // ลองที่อยู่ต่อไปถ้าการเชื่อมต่อล้มเหลว
                continue;
            }
        }
    }

    Err(format!("Failed to connect to any address for {}", addr))
}
```

คำอธิบายโค้ด:
1. `tokio::net::lookup_host` ทำการ DNS lookup แบบ asynchronous เพื่อได้รายการที่อยู่ IP ที่เกี่ยวข้องกับโดเมน
2. ผลลัพธ์ถูกแปลงเป็น `Vec` และกรองเฉพาะที่อยู่ IPv4 ด้วย `filter(|a| a.is_ipv4())`
3. หากไม่มีที่อยู่ IPv4 เหลืออยู่ จะคืนค่าข้อผิดพลาดทันที
4. ลูปผ่านแต่ละที่อยู่และพยายามสร้างการเชื่อมต่อ TCP ด้วย `TcpStream::connect`
5. เมื่อการเชื่อมต่อสำเร็จ เราตั้งค่า `TCP_NODELAY` เป็น `true` เพื่อปิด Nagle's algorithm ซึ่งทำให้ข้อมูลเล็กถูกส่งทันทีโดยไม่รอให้ buffer เต็ม เหมาะสำหรับแอปพลิเคชันที่ต้องการความหน่วงต่ำเช่น WebSocket proxy
6. หากการตั้งค่า `TCP_NODELAY` ล้มเหลว จะคืนค่าข้อผิดพลาด
7. หากการเชื่อมต่อล้มเหลวกับที่อยู่หนึ่ง จะลองที่อยู่ถัดไปในรายการ
8. หากไม่สามารถเชื่อมต่อได้กับที่อยู่ใดเลย จะคืนค่าข้อผิดพลาดที่บ่งบอกว่าล้มเหลวทั้งหมด

การออกแบบเช่นนี้ทำให้ proxy มีความทนทานต่อการเปลี่ยนแปลงของ DNS และสามารถทำงานได้ในสภาพแวดล้อมที่มีหลายที่อยู่ IP (เช่น load balancing)

## Bidirectional Pump — แนวคิด

ใน WebSocket proxy เราต้องส่งข้อมูลสองทิศทางพร้อมกัน:
- จาก WebSocket ไปยัง TCP (ws → tcp): ข้อความที่ได้รับจากไคลเอนต์ WebSocket จะถูกส่งไปยังเซิร์ฟเวอร์ TCP เป้าหมาย
- จาก TCP ไปยัง WebSocket (tcp → ws): ข้อมูลที่ได้รับจากเซิร์ฟเวอร์ TCP จะถูกส่งกลับไปยังไคลเอนต์ WebSocket

เหตุผลที่ต้องทำสองทิศทางพร้อมกันก็คือการสื่อสารผ่าน proxy มักเป็นแบบ duplex หมายความว่าทั้งไคลเอนต์และเซิร์ฟเวอร์สามารถส่งข้อมูลได้ตลอดเวลาโดยไม่ต้องรอให้อีกฝ่ายส่งเสร็จก่อน หากเราทำแบบ sequential (ทำทิศทางหนึ่งให้เสร็จก่อนค่อยทำอีกทิศทาง) จะทำให้เกิดความหน่วงและอาจทำให้ข้อมูลค้างอยู่ใน buffer ไม่ถูกส่งออกไปทันที

ใน Rust กับ Tokio เราใช้ `tokio::select!` macro เพื่อรอเหตุการณ์จากหลายแหล่งพร้อมกัน มันจะทำงานเมื่อใดก็ตามที่หนึ่งในหลายๆ ฟิวเจอร์ที่เรากำหนดพร้อมที่จะทำงาน จากนั้นเราจะรันฟิวเจอร์นั้นและทำซ้ำไปเรื่อยๆ จนกว่าจะมีเงื่อนไขให้หยุด

ตัวอย่างการใช้งานคร่าวๆ:
```rust
tokio::select! {
    result = ws_to_tcp => {
        // จัดการเมื่อทิศทาง ws→tcp เสร็จสิ้น
    }
    result = tcp_to_ws => {
        // จัดการเมื่อทิศทาง tcp→ws เสร็จสิ้น
    }
}
```

ด้วยวิธีนี้เราสามารถจัดการการไหลของข้อมูลสองทิศทางได้อย่างมีประสิทธิภาพโดยไม่บล็อกซึ่งกันและกัน

## WS → TCP Direction

ทิศทางจาก WebSocket ไปยัง TCP มีหน้าที่รับข้อความจาก WebSocket receiver (`ws_rx`) และเขียนไปยัง TCP writer (`tcp_write`) เราสนใจเฉพาะข้อความประเภท binary เท่านั้นเพราะโปรโตคอลของเรา (roBrowser) ใช้ binary frame ในการส่งข้อมูล

```rust
let ws_to_tcp = async {
    while let Some(msg) = ws_rx.next().await {
        match msg {
            Ok(Message::Binary(data)) => {
                // เขียนข้อมูล binary ลงใน TCP stream
                if let Err(e) = tcp_write.write_all(&data).await {
                    // หากเกิดข้อผิดพลาดในการเขียน ให้ออกจากลูป
                    break;
                }
            }
            Ok(Message::Close(_)) => {
        // ได้รับสัญญาณปิดการเชื่อมต่อจาก WebSocket
                break;
            }
            _ => {
                // ละเลยข้อความประเภทอื่นๆ (เช่น text, ping, pong)
            }
        }
    }
};
```

คำอธิบาย:
- เราใช้ `ws_rx.next().await` เพื่อรับข้อความถัดจาก WebSocket stream แบบ non-blocking
- เมื่อได้รับ `Message::Binary(data)` เราจะเขียนข้อมูลทั้งหมดลงใน `tcp_write` ด้วย `write_all` ซึ่งจะพยายามเขียนข้อมูลจนหมดก่อนจะคืนค่า
- หากการเขียนล้มเหลว (เช่น เซิร์ฟเวอร์ TCP ปิดการเชื่อมต่อ) เราจะออกจากลูปทันที
- หากได้รับ `Message::Close` เราจะออกจากลูปเพื่อจบการทำงานของทิศทางนี้
- ข้อความประเภทอื่นๆ (เช่น text, ping, pong) จะถูกละเลยเพราะไม่เกี่ยวข้องกับการส่งข้อมูลต้นทางในโปรโตคอลของเรา

การออกแบบให้รับเฉพาะ binary frame ช่วยให้เรามั่นใจว่าข้อมูลที่ส่งไปยัง TCP เซิร์ฟเวอร์นั้นเป็นข้อมูลต้นทางที่ไม่ถูกดัดแปลง ซึ่งสำคัญต่อการทำงานของโปรโตคอลเช่น roBrowser ที่คาดหวังข้อมูลไบนารีที่สมบูรณ์

## TCP → WS Direction

ทิศทางจาก TCP ไปยัง WebSocket มีความซับซ้อนมากขึ้นเล็กน้อยเพราะเราต้องจัดการกับการอ่านข้อมูลจาก TCP stream ที่อาจมาเป็นชิ้นส่วน fragmented เราใช้ `BytesMut` เป็น buffer แบบไดนามิกเพื่อสะสมข้อมูลที่อ่านมาได้จนกว่าจะมีข้อมูลพร้อมส่งเป็น WebSocket binary frame

```rust
let tcp_to_ws = async {
    let mut buf = BytesMut::with_capacity(65536);
    loop {
        // ตรวจสอบและขยาย buffer หากความจุเหลือน้อยเกินไป
        if buf.capacity() < 4096 {
            buf.reserve(65536);
        }
        match tcp_read.read_buf(&mut buf).await {
            Ok(0) => {
                // อ่านได้ 0 ไบต์ หมายถึงการเชื่อมต่อ TCP ถูกปิดอย่างปกติ
                let _ = ws_tx.send(Message::Close(None)).await;
                break;
            }
            Ok(n) => {
                // อ่านได้ n ไบต์ (> 0)
                // แบ่งข้อมูลที่อ่านได้ออกจาก buffer เป็น independent slice
                let data = buf.split().freeze();
                // พยายามส่งข้อมูลเป็น WebSocket binary frame
                if ws_tx.send(Message::Binary(data)).await.is_err() {
                    // หากส่งไม่สำเร็จ (เช่น WebSocket ถูกปิด) ให้ออกจากลูป
                    break;
                }
                // วนลูปต่อเพื่ออ่านข้อมูลเพิ่มเติม
            }
            Err(e) => {
                // เกิดข้อผิดพลาดในการอ่านจาก TCP
                break;
            }
        }
    }
};
```

คำอธิบายอย่างละเอียด:
1. เราเริ่มต้นด้วยการสร้าง `BytesMut` ที่มีความจุเริ่มต้น 64 KiB (65536 ไบต์) เพื่อลดการจัดสรรหน่วยความจำบ่อยครั้ง
2. ในแต่ละรอบของลูป เราตรวจสอบว่าความจุที่เหลือใน buffer น้อยกว่า 4 KiB หรือไม่ หากน้อยกว่าเราจะสำรองพื้นที่เพิ่มอีก 64 KiB ด้วย `reserve` เพื่อให้มีพื้นที่เพียงพอสำหรับการอ่านครั้งต่อไป
3. เราเรียก `tcp_read.read_buf(&mut buf).await` เพื่ออ่านข้อมูลจาก TCP stream เข้าไปใน buffer โดยไม่คัดลอกข้อมูล (zero-copy) เนื่องจาก `read_buf` จะเขียนข้อมูลโดยตรงเข้าไปใน buffer ที่เราให้มา
4. หากการอ่านได้ 0 ไบต์ (`Ok(0)`) หมายถึงการเชื่อมต่อ TCP ถูกปิดโดยเซิร์ฟเวอร์ปลายทางอย่างปกติ เราจึงส่งสัญญาณ `Message::Close` ไปยัง WebSocket เพื่อแจ้งให้ไคลเอนต์ทราบว่าการเชื่อมต่อกำลังจะปิด จากนั้นออกจากลูป
5. หากการอ่านได้ข้อมูล (`Ok(n)` โดยที่ n > 0):
   - เราใช้ `buf.split()` เพื่อแบ่งข้อมูลทั้งหมดที่อยู่ใน buffer ออกเป็นสองส่วน: ส่วนที่ถูกแบ่งออก (ซึ่งมีข้อมูลที่เราอ่านได้ในรอบนี้) และส่วนที่เหลืออยู่ใน buffer เดิม
   - จากนั้นเราเรียก `.freeze()` บนส่วนที่ถูกแบ่งออกเพื่อแปลงมันเป็น `Bytes` ที่ไม่สามารถเปลี่ยนแปลงได้ ซึ่งเหมาะสำหรับการส่งผ่านไปยังส่วนอื่นของโปรแกรมโดยไม่ต้องกังวลว่าจะถูกแก้ไข
   - สุดท้ายเราพยายามส่ง `Bytes` นี้เป็น WebSocket binary frame ด้วย `ws_tx.send(Message::Binary(data)).await`
   - หากการส่งล้มเหลว (คืนค่า `Err`) เราจะออกจากลูปทันทีเพราะอาจหมายถึงการเชื่อมต่อ WebSocket มีปัญหา
6. หากเกิดข้อผิดพลาดในการอ่านจาก TCP (`Err(e)`) เราจะออกจากลูปทันที

การใช้ `BytesMut` ร่วมกับ `split()` และ `freeze()` ช่วยให้เราสามารถจัดการ buffer ได้อย่างมีประสิทธิภาพโดยไม่ต้องคัดลอกข้อมูลหลายครั้ง เราอ่านข้อมูลเข้าไปใน buffer เดียว เมื่อมีข้อมูลพร้อมส่งเราจะแบ่งออกมาเป็นชิ้นอิสระแล้วส่งออกไป ทำให้เหลือพื้นที่ใน buffer เดิมสำหรับอ่านข้อมูลครั้งต่อไปได้ทันที โดยไม่ต้องเคลียร์หรือย้ายข้อมูล

## Stream Splitting

ทำไมเราต้องแยก TCP stream ออกเป็นสองส่วน (`tcp.into_split()`) แทนที่จะใช้ stream เดียวกันสำหรับการอ่านและเขียน? คำตอบอยู่ที่ลักษณะของ `TcpStream` ใน Tokio ที่เมื่อเราใช้วิธีการอ่านและเขียนบน stream เดียวกันจากงานหลายงานพร้อมกัน จะเกิดการแย่งชิงทรัพยากร (contention) บน BiLock ภายใน ซึ่งทำให้ประสิทธิภาพลดลง

เมื่อเราเรียก `tcp.into_split()` เราจะได้คู่ `(ReadHalf, WriteHalf)` ซึ่งแต่ละส่วนสามารถถูกใช้งานโดยงานต่างๆ ได้อย่างอิสระโดยไม่ต้องแย่งชิงล็อคเดียวกัน นี่เป็นสิ่งสำคัญอย่างยิ่งในสถานการณ์เช่น proxy ที่เรามีงานอ่านจาก TCP (เพื่อส่งไปยัง WebSocket) และงานเขียนไปยัง TCP (จาก WebSocket) ทำงานพร้อมกันตลอดเวลา

โดยการแยก stream เราจึงหลีกเลี่ยงปัญหาประสิทธิภาพที่อาจเกิดขึ้นจากการแย่งชิงล็อค และทำให้การอ่านและเขียนสามารถทำงานได้อย่างเต็มความเร็วของการเชื่อมต่อ TCP โดยไม่มีการหน่วงเหนี่ยวที่ไม่จำเป็น

## TLS ด้วย rustls

การรองรับ TLS ใน wsProxy ทำได้โดยการใช้ไลบรารี `rustls` ซึ่งเป็นไลบรารี TLS ที่ทันสมัยและปลอดภัยเขียนด้วย Rust อย่างบริสุทธิ์ เราไม่ใช้ OpenSSL เพื่อหลีกเลี่ยงปัญหาการจัดการหน่วยความจำและความซับซ้อนในการสร้าง

ฟังก์ชัน `start_tls` มีหน้าที่เพิ่มการเข้ารหัส TLS ให้กับการเชื่อมต่อ TCP ที่มีอยู่แล้ว มันทำงานโดยการโหลดใบรับรองและคีย์ส่วนตัวจากไฟล์ PEM สร้างคอนฟิกูเรชัน `rustls` จากนั้นจึงทำการ handshake TLS กับเซิร์ฟเวอร์ปลายทาง

```rust
pub async fn start_tls(stream: TcpStream, domain: &str) -> Result<TlsStream<TcpStream>, String> {
    // โหลดใบรับรองจากไฟล์ PEM
    let certs = rustls_pemfile::certs(&mut std::fs::File::open("certs/server.crt")
        .map_err(|e| format!("Failed to open certificate file: {}", e))?)
        .collect::<Result<Vec<_>, _>>()
        .map_err(|e| format!("Failed to parse certificate: {}", e))?;

    // โหลดคีย์ส่วนตัวจากไฟล์ PEM
    let key = rustls_pemfile::pkcs8_private_keys(&mut std::fs::File::open("certs/server.key")
        .map_err(|e| format!("Failed to open key file: {}", e))?)
        .remove(0)
        .map_err(|e| format!("Failed to parse private key: {}", e))?;

    // สร้างคอนฟิกูเรชันสำหรับไคลเอนต์ TLS
    let config = rustls::ClientConfig::builder()
        .with_safe_defaults()
        .with_root_certificates(certs)
        .with_single_cert(vec![certs[0].clone()], key)
        .map_err(|e| format!("Failed to build TLS config: {}", e))?;

    // ทำการ handshake TLS กับเซิร์ฟเวอร์ปลายทางโดยใช้ชื่อโดเมนสำหรับ SNI
    let connector = rustls::TokioTlsConnector::from(Arc::new(config));
    let tls_stream = connector
        .connect(domain, stream)
        .await
        .map_err(|e| format!("TLS handshake failed: {}", e))?;

    Ok(tls_stream)
}
```

คำอธิบาย:
1. เราโหลดใบรับรองจากไฟล์ `certs/server.crt` โดยใช้ `rustls_pemfile::certs` ซึ่งคืนค่าอิตอเรเตอร์ของใบรับรอง เราเก็บรวบรวมเข้าสู่เวกเตอร์และจัดการกับข้อผิดพลาดที่อาจเกิดขึ้นจากการเปิดไฟล์หรือการแปลงรูปแบบ
2. เช่นเดียวกันเราโหลดคีย์ส่วนตัวจากไฟล์ `certs/server.key` โดยใช้ `rustls_pemfile::pkcs8_private_keys` เราสมมติว่ามีคีย์ส่วนตัวเพียงหนึ่งเดียวในไฟล์จึงใช้ `.remove(0)` เพื่อดึงออกมา
3. เราสร้าง `rustls::ClientConfig` โดย:
   - ใช้ค่าตั้งค่าที่ปลอดภัยเป็นค่าเริ่มต้นด้วย `with_safe_defaults()`
   - เพิ่มใบรับรองรูทที่เชื่อถือได้ด้วย `with_root_certificates` (ในกรณีนี้เราใช้ใบรับรองเดียวกันกับที่เราโหลดมาเป็นรูท ซึ่งเหมาะสำหรับการทดสอบหรือการใช้ใบรับรองที่ลงชื่อเอง)
   - กำหนดใบรับรองและคีย์ส่วนตัวของไคลเอนต์ด้วย `with_single_cert`
4. สร้าง `TokioTlsConnector` จากคอนฟิกูเรชันที่ได้มา
5. ทำการเชื่อมต่อ TLS โดยเรียก `connector.connect(domain, stream)` โดยพารามิเตอร์แรกคือชื่อโดเมนที่ใช้สำหรับ Server Name Indication (SNI) และพารามิเตอร์ที่สองคือ TCP stream ที่มีอยู่แล้ว
6. หาก handshake สำเร็จ เราจะได้ `TlsStream<TcpStream>` ซึ่งห่อหุ้ม TCP stream เดิมและให้อินเทอร์เฟซเดียวกันสำหรับการอ่านและเขียน แต่ข้อมูลจะถูกเข้ารหัสและถอดรหัสโดยอัตโนมัติผ่าน TLS

ในส่วนของเซิร์ฟเวอร์ (เมื่อ wsProxy ทำหน้าที่เป็นเซิร์ฟเวอร์ WebSocket ที่มี TLS) เราใช้ความสามารถที่ติดมากับ `axum-server` ซึ่งรองรับ TLS โดยตรงผ่านการระบุ `tls_config` ในการสร้างเซิร์ฟเวอร์ ทำให้เราไม่ต้องจัดการ handshake TLS ด้วยตัวเองในระดับแอปพลิเคชัน

## Graceful Shutdown

การปิดการทำงานอย่างสง่างาม (graceful shutdown) เป็นสิ่งสำคัญสำหรับเซิร์ฟเวอร์ผลิตเพื่อให้แน่ใจว่าการเชื่อมต่อที่มีอยู่ทั้งหมดได้รับการจัดการอย่างเหมาะสมก่อนที่กระบวนการจะสิ้นสุดลง เราจัดการกับสัญญาณสองประเภท:
- `SIGINT` (จากการกด Ctrl+C ในเทอร์มินัล)
- `SIGTERM` (สัญญาณปิดการทำงานมาตรฐานจากระบบปฏิบัติการหรือผู้จัดการกระบวนการเช่น Kubernetes)

เราใช้ `tokio::select!` เพื่อรอทั้งการทำงานของเซิร์ฟเวอร์และสัญญาณเหล่านี้พร้อมกัน เมื่อรับสัญญาณใดสัญญาณหนึ่ง เราจะเริ่มกระบวนการปิดการทำงานโดย:
1. หยุดการรับการเชื่อมต่อใหม่
2. รอให้การเชื่อมต่อที่มีอยู่ทั้งหมดเสร็จสิ้นภายในระยะเวลาที่กำหนด (เช่น 30 วินาที)
3. หากเกินเวลาที่กำหนด เราจะบังคับปิดการเชื่อมต่อที่เหลืออยู่และออกจากโปรแกรม

```rust
#[tokio::main]
async fn main() {
    // ... การตั้งค่าเซิร์ฟเวอร์และตัวแปรต่างๆ

    // สร้างสัญญาณสำหรับการปิดการทำงาน
    let shutdown_signal = async {
        tokio::select! {
            _ = tokio::signal::ctrl_c() => {
                println!("รับสัญญาณ Ctrl+C เริ่มการปิดการทำงาน");
            }
            _ = tokio::signal::sigterm() => {
                println!("รับสัญญาณ SIGTERM เริ่มการปิดการทำงาน");
            }
        }
    };

    // รอทั้งการทำงานของเซิร์ฟเวอร์และสัญญาณปิดการทำงาน
    tokio::select! {
        _ = server_future => {
            println!("เซิร์ฟเวอร์หยุดทำงานโดยไม่มีสัญญาณ");
        }
        _ = shutdown_signal => {
            // เริ่มกระบวนการปิดการทำงานอย่างสง่างาม
            println!("เริ่มการปิดการทำงานอย่างสง่างาม...");
            // ที่นี่เราจะหยุดการรับการเชื่อมต่อใหม่และรอให้การเชื่อมต่อที่มีอยู่เสร็จสิ้น
            // ตัวอย่างการใช้ shutdown_timeout หรือวิธีการที่คล้ายกันขึ้นกับเฟรมเวิร์กที่ใช้
            // สำหรับตัวอย่างนี้เราสมมติว่ามีฟังก์ชัน graceful_shutdown ที่จัดการขั้นตอนเหล่านี้
            if let Err(e) = graceful_shutdown().await {
                eprintln!("เกิดข้อผิดพลาดระหว่างการปิดการทำงาน: {}", e);
            }
        }
    }
}
```

ในการใช้งานจริงกับ `axum` เราอาจใช้วิธีการเช่น:
```rust
let server = Server::bind(&addr).serve(app.into_make_service());
let graceful = server.with_graceful_shutdown(shutdown_signal);
if let Err(e) = graceful.await {
    eprintln!("เซิร์ฟเวอร์ทำงานผิดพลาด: {}", e);
}
```

โดยที่ `shutdown_signal` คือฟิวเจอร์ที่จะเสร็จสมบูรณ์เมื่อได้รับสัญญาณปิดการทำงาน วิธีนี้ทำให้ `axum` หยุดรับการเชื่อมต่อใหม่และรอให้การเชื่อมต่อที่มีอยู่เสร็จสิ้นก่อนที่จะปิดตัวลง

## Testing

การทดสอบเป็นส่วนสำคัญที่ช่วยให้มั่นใจว่าโค้ดของเราทำงานได้อย่างถูกต้องและสามารถปรับปรุงหรือรีฟักเตอร์ได้โดยไม่กลัวว่าจะทำลายฟังก์ชันการทำงานที่มีอยู่ ใน wsProxy เรามีการทดสอบสองระดับ:
1. **Unit Tests**: ทดสอบฟังก์ชันและเมธอดแต่ละตัวอย่างแยกจากกัน
2. **Integration Tests**: ทดสอบการทำงานร่วมกันของหลายส่วนในสภาพแวดล้อมที่ใกล้เคียงกับการใช้งานจริง

### Unit Tests ตัวอย่าง

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tokio::net::TcpListener;

    #[tokio::test]
    async fn test_connect_tcp_to_echo() {
        // เริ่มเซิร์ฟเวอร์ echo แบบง่ายๆ บนที่อยู่สุ่ม
        let (handle, echo_addr) = echo_server("127.0.0.1:0").await;
        let addr = echo_addr.to_string();

        // ทดสอบการเชื่อมต่อ TCP ไปยังเซิร์ฟเวอร์ echo
        let mut stream = connect_tcp(&addr).await.expect("ควรเชื่อมต่อ TCP สำเร็จ");
        
        // ส่งข้อมูลทดสอบ
        let test_data = b"hello wsProxy";
        stream.write_all(test_data).await.expect("ควรเขียนข้อมูลสำเร็จ");

        // อ่านข้อมูลที่สะท้อนกลับมา
        let mut buffer = Vec::new();
        stream.read_to_end(&mut buffer).await.expect("ควรอ่านข้อมูลสำเร็จ");
        
        // ตรวจสอบว่าข้อมูลที่ได้รับตรงกับที่ส่งไป
        assert_eq!(buffer, test_data);
        
        // ทำความสะอาดโดยปิดเซิร์ฟเวอร์ทดสอบ
        handle.abort();
    }

    #[tokio::test]
    async fn test_validate_target() {
        // ทดสอบการตรวจสอบที่อยู่เป้าหมายว่าถูกต้องหรือไม่
        assert!(validate_target("example.com:80").is_ok());
        assert!(validate_target("192.168.1.1:443").is_ok());
        assert!(validate_target("[2001:db8::1]:8080").is_ok()); // IPv6
        
        // ที่อยู่ที่ไม่ถูกต้อง
        assert!(validate_target("invalid_hostname!!!:80").is_err());
        assert!(validate_target("example.com:99999").is_err()); // พอร์ตเกินขอบเขต
    }
}

// ฟังก์ชันช่วยสำหรับสร้างเซิร์ฟเวอร์ echo แบบง่ายๆ สำหรับการทดสอบ
async fn echo_server(addr: &str) -> (tokio::task::JoinHandle<()>, SocketAddr) {
    let listener = TcpListener::bind(addr).await.expect("ควรผูกที่อยู่สำเร็จ");
    let addr = listener.local_addr().expect("ควรได้ที่อยู่ท้องถิ่น");
    let handle = tokio::spawn(async move {
        loop {
            match listener.accept().await {
                Ok((mut socket, _)) => {
                    tokio::spawn(async move {
                        let mut buf = vec![0; 1024];
                        loop {
                            let n = match socket.read(&mut buf).await {
                                Ok(0) => return, // การเชื่อมต่อถูกปิด
                                Ok(n) => n,
                                Err(e) => {
                                    eprintln!("เกิดข้อผิดพลาดในการอ่าน: {}", e);
                                    return;
                                }
                            };
                            if let Err(e) = socket.write_all(&buf[..n]).await {
                                eprintln!("เกิดข้อผิดพลาดในการเขียน: {}", e);
                                return;
                            }
                        }
                    });
                }
                Err(e) => {
                    eprintln!("เกิดข้อผิดพลาดในการยอมรับการเชื่อมต่อ: {}", e);
                    break;
                }
            }
        }
    });
    (handle, addr)
}
```

คำอธิบายการทดสอบ:
- `test_connect_tcp_to_echo`: เริ่มต้นโดยการสร้างเซิร์ฟเวอร์ echo แบบง่ายๆ ที่ฟังบนที่อยู่สุ่ม (`127.0.0.1:0` หมายถึงให้ระบบเลือกพอร์ตว่าง) จากนั้นทดสอบการเชื่อมต่อ TCP ไปยังที่อยู่นั้นโดยใช้ฟังก์ชัน `connect_tcp` ที่เราต้องการทดสอบ หลังจากเชื่อมต่อสำเร็จเราส่งข้อความทดสอบและตรวจสอบว่าข้อมูลที่ได้รับกลับมาตรงกับที่ส่งไป
- `test_validate_target`: ทดสอบฟังก์ชัน `validate_target` (ซึ่งเราสมมติว่ามีอยู่ในโค้ดจริง) ด้วยกรณีต่างๆ ทั้งที่อยู่ที่ถูกต้องและไม่ถูกต้อง เพื่อให้มั่นใจว่าฟังก์ชันสามารถตรวจจับที่อยู่ที่ผิดรูปแบบได้อย่างถูกต้อง

### Integration Tests ตัวอย่าง

สำหรับการทดสอบแบบ integration เราอาจต้องการทดสอบการทำงานทั้งหมดของ proxy ตั้งแต่การรับการเชื่อมต่อ WebSocket ไปจนถึงการสื่อสารกับเซิร์ฟเวอร์ TCP เป้าหมายและกลับมา ซึ่งอาจต้องใช้เฟรมเวิร์กทดสอบที่สามารถจำลองไคลเอนต์ WebSocket และเซิร์ฟเวอร์ TCP ได้

ตัวอย่างการตั้งค่า integration test อาจมีลักษณะดังนี้ (โค้ดจริงอาจซับซ้อนกว่านี้ขึ้นกับความต้องการ):

```rust
#[tokio::test]
async fn test_websocket_to_tcp_echo() {
    // เริ่มเซิร์ฟเวอร์ TCP echo ที่อยู่เบื้องหลัง proxy
    let tcp_handle = start_tcp_echo_server().await;
    
    // เริ่ม wsProxy เซิร์ฟเวอร์ที่ชี้ไปยังเซิร์ฟเวอร์ TCP echo ที่เราเริ่มต้นไว้
    let proxy_handle = start_wsproxy_server(tcp_echo_addr).await;
    
    // เชื่อมต่อเป็นไคลเอนต์ WebSocket ไปยัง wsProxy
    let mut ws_client = connect_wsclient(proxy_addr).await?;
    
    // ส่งข้อความ binary ผ่าน WebSocket
    let test_msg = b"integration test data";
    ws_client.send(Message::Binary(test_msg.to_vec())).await?;
    
    // รับข้อความตอบกลับจาก WebSocket ควรเป็นข้อมูลเดียวกันที่ส่งไป
    if let Some(Ok(Message::Binary(received))) = ws_client.recv().await {
        assert_eq!(received, test_msg);
    } else {
        panic!("ไม่ได้รับข้อความ binary ที่คาดหวัง");
    }
    
    // ทำความสะอาด
    ws_client.close().await?;
    proxy_handle.abort();
    tcp_handle.abort();
}
```

การทดสอบเหล่านี้ช่วยให้เราตรวจสอบได้ว่า:
1. ฟังก์ชัน `connect_tcp` ทำงานได้อย่างถูกต้องในการสร้างการเชื่อมต่อ TCP
2. กลไกการแบ่งปันทิศทางข้อมูล (bidirectional pump) สามารถส่งข้อมูลทั้งสองทิศทางได้อย่างถูกต้อง
3. การจัดการข้อผิดพลาดและการปิดการเชื่อมต่อทำงานได้ตามที่คาดหวัง
4. การรองรับ TLS (ถ้ามีการตั้งค่า) ทำงานได้อย่างถูกต้อง
5. การตั้งค่าเซิร์ฟเวอร์และการทำงานโดยรวมทำงานได้อย่างถูกต้องในสภาพแวดล้อมที่ใกล้เคียงกับการใช้งานจริง

การมีการทดสอบที่ครอบคลุมช่วยให้เราสามารถรีฟักเตอร์โค้ดได้อย่างมั่นใจและลดโอกาสที่จะนำบั๊กเข้ามาโดยไม่ได้ตั้งใจ

## Docker Deployment

การจัดส่ง wsProxy ด้วย Docker ทำให้ง่ายต่อการนำไปใช้งานในสภาพแวดล้อมต่างๆ โดยไม่ต้องกังวลเกี่ยวกับการติดตั้งขึ้นต่อกันหรือความแตกต่างของระบบปฏิบัติการ เราใช้เทคนิค multi-stage build เพื่อสร้างภาพที่มีขนาดเล็กและปลอดภัย โดยแยกขั้นตอนการสร้าง (build) ออกจากขั้นตอนการรันไทม์ (runtime)

### Containerfile

```dockerfile
# ขั้นตอนการสร้าง (builder stage)
FROM rust:1-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

# ขั้นตอนการรันไทม์ (runtime stage)
FROM debian:bookworm-slim
COPY --from=builder /app/target/release/rs-wsProxy /usr/local/bin/
EXPOSE 5999
CMD ["rs-wsProxy"]
```

คำอธิบายขั้นตอนโดยละเอียด:
1. **ขั้นตอนการสร้าง (builder stage)**:
   - เริ่มต้นจากภาพ `rust:1-bookworm` ซึ่งมี Rust toolchain ที่ติดตั้งมาแล้วพร้อมใช้งาน
   - ตั้งไดเรกทอรีการทำงานเป็น `/app`
   - คัดลอกไฟล์โครงการทั้งหมดเข้าไปในภาพด้วย `COPY . .`
   - รัน `cargo build --release` เพื่อสร้างไบนารีตัวปล่อยในโหมดปล่อยที่ปรับให้เหมาะสมที่สุด

2. **ขั้นตอนการรันไทม์ (runtime stage)**:
   - เริ่มต้นจากภาพ `debian:bookworm-slim` ซึ่งเป็นภาพ Debian ที่มีขนาดเล็กและมีเฉพาะสิ่งจำเป็นพื้นฐาน
   - คัดลอกไบนารีที่สร้างได้จากขั้นตอน builder ไปยังตำแหน่ง `/usr/local/bin/` ในภาพรันไทม์ด้วย `COPY --from=builder /app/target/release/rs-wsProxy /usr/local/bin/`
   - เปิดเผยพอร์ต 5999 ซึ่งเป็นพอร์ตเริ่มต้นที่ wsProxy ใช้ฟังการเชื่อมต่อเข้ามา
   - กำหนดคำสั่งเริ่มต้นเมื่อรันคอนเทนเนอร์เป็น `rs-wsProxy` ซึ่งจะเรียกใช้ไบนารีที่เราคัดลอกมา

ข้อดีของวิธีการนี้คือ ภาพสุดท้ายจะไม่มีเครื่องมือสร้าง Rust หรือไฟล์ต้นทางใดๆ เลย ทำให้มีขนาดเล็กและลดพื้นที่ผิวที่อาจถูกโจมตี (attack surface) เนื่องจากมีเพียงไบนารีและไลบรารีระบบที่จำเป็นเท่านั้น

### docker-compose.yaml สำหรับการพัฒนาและทดสอบ

สำหรับการพัฒนาและทดสอบในเครื่อง local เราสามารถใช้ docker-compose เพื่อกำหนดบริการต่างๆ ได้อย่างง่ายดาย ตัวอย่างต่อไปนี้แสดงให้เห็นถึงการรัน wsProxy พร้อมกับเซิร์ฟเวอร์ทดสอบแบบ echo:

```yaml
version: '3.8'

services:
  wsproxy:
    build: .
    ports:
      - "5999:5999"
    environment:
      - LOG_LEVEL=info
      - TARGET_HOST=echo-server
      - TARGET_PORT=7
    depends_on:
      - echo-server

  echo-server:
    image: nginx:alpine
    command: ["/bin/sh", "-c", "while true; do echo -e 'HTTP/1.1 200 OK\r\n\r\nHello from echo server' | nc -l -p 7; done"]
    ports:
      - "7:7"
```

คำอธิบาย:
- บริการ `wsproxy` สร้างภาพจาก Dockerfile ในไดเรกทอรีปัจจุบันและแมปพอร์ต 5999 ของคอนเทนเนอร์ไปยังพอร์ต 5999 ของโฮสต์
- ตัวแปรสภาพแวดล้อมเช่น `LOG_LEVEL`, `TARGET_HOST`, และ `TARGET_PORT` ถูกส่งไปยังคอนเทนเนอร์เพื่อตั้งค่าการทำงานของ wsProxy (สมมติว่า wsProxy อ่านการตั้งค่าจากตัวแปรสภาพแวดล้อม)
- บริการ `echo-server` ใช้ภาพ `nginx:alpine` และรันคำสั่ง shell ที่สร้างเซิร์ฟเวอร์ echo แบบง่ายๆ บนพอร์ต 7 โดยใช้ `netcat` (nc)
- `depends_on` ทำให้แน่ใจว่าเซิร์ฟเวอร์ echo จะเริ่มต้นก่อนที่ wsProxy จะพยายามเชื่อมต่อไปยังมัน

ด้วยวิธีนี้ นักพัฒนาสามารถรัน `docker-compose up` เพื่อเริ่มต้นทั้งสองบริการและทดสอบการทำงานของ wsProxy ได้ทันทีโดยไม่ต้องติดตั้งอะไรเพิ่มเติมบนเครื่องโฮสต์

## Kubernetes Deployment

สำหรับการจัดส่งในสภาพแวดล้อมผลิตที่ต้องการความสามารถในการขยายขนาด การจัดการ และความทนทานต่อความล้มเหลว เราสามารถใช้ Kubernetes ได้ แม้ว่าการตั้งค่าเต็มรูปแบบอาจซับซ้อน แต่เราจะแสดงองค์ประกอบพื้นฐานที่จำเป็นสำหรับการรัน wsProxy ในคลัสเตอร์ Kubernetes

### Deployment

ไฟล์ `deploy/k8s/deployment.yaml` กำหนดว่าจะรันกี่รีพอร์ติกของ wsProxy และตั้งค่าต่างๆ เช่น ภาพคอนเทนเนอร์ ตัวแปรสภาพแวดล้อม และทรัพยากรที่ต้องการ

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wsproxy-deployment
  labels:
    app: wsproxy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: wsproxy
  template:
    metadata:
      labels:
        app: wsproxy
    spec:
      containers:
        - name: wsproxy
          image: bouroo/rs-wsproxy:latest
          ports:
            - containerPort: 5999
          env:
            - name: LOG_LEVEL
              value: "info"
            - name: TARGET_HOST
              value: "backend-service"
            - name: TARGET_PORT
              value: "8080"
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
```

คำอธิบายสำคัญ:
- `replicas: 3` ระบุว่าต้องการให้มีรีพอร์ติกของ wsProxy ทำงานอยู่ 3 ตัวพร้อมกันเพื่อความทนทานต่อความล้มเหลวและความสามารถในการรับโหลด
- `selector` และ `template.metadata.labels` ใช้เพื่อเชื่อมโยง Deployment กับ Pod ที่มันจัดการ
- ภาพคอนเทนเนอร์ที่ใช้คือ `bouroo/rs-wsproxy:latest` (สมมติว่ามีการส่งภาพไปยัง registry แล้ว)
- เปิดเผยพอร์ตคอนเทนเนอร์ 5999 ซึ่งเป็นพอร์ตที่ wsProxy ฟังการเชื่อมต่อเข้ามา
- ตัวแปรสภาพแวดล้อมตั้งค่าระดับการบันทึก โฮสต์และพอร์ตของเซิร์ฟเวอร์เป้าหมายที่ proxy ควรเชื่อมต่อไปยัง
- ส่วน `resources` กำหนดขั้นต่ำและสูงสุดของทรัพยากรที่คอนเทนเนอร์สามารถใช้ได้ ช่วยให้ Kubernetes จัดสรรทรัพยากรได้อย่างเหมาะสมและป้องกันไม่ให้คอนเทนเนอร์ใช้ทรัพยากรมากเกินไป

### Service

ไฟล์ `deploy/k8s/service.yaml` กำหนดวิธีการเข้าถึง wsProxy จากภายในหรือภายนอกคลัสเตอร์ Kubernetes ในตัวอย่างนี้เราใช้ `LoadBalancer` เพื่อให้ได้ที่อยู่ IP สาธารณะ (หากรันบนคลาวด์ที่สนับสนุน) หรือสามารถเปลี่ยนเป็น `NodePort` หรือ `ClusterIP` ได้ตามความต้องการ

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wsproxy-service
  labels:
    app: wsproxy
spec:
  selector:
    app: wsproxy
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5999
  type: LoadBalancer
```

คำอธิบาย:
- `selector` ตรงกับ label ของ Pod ที่สร้างจาก Deployment ทำให้บริการนี้ส่งต่อการเชื่อมต่อไปยัง Pod ของ wsProxy
- บริการฟังที่พอร์ต 80 และส่งต่อการเชื่อมต่อไปยังพอร์ต 5999 ของ Pod (`targetPort`)
- ประเภท `LoadBalancer` ทำให้ผู้ให้บริการคลาวด์สร้างโหลดบาลานเซอร์ภายนอกเพื่อกระจายการจราจรไปยังรีพอร์ติกทั้งหมด

### ConfigMap (ตัวเลือก)

หากเราต้องการแยกการตั้งค่าออกจากภาพคอนเทนเนอร์ เราสามารถใช้ ConfigMap ได้ ตัวอย่างเช่นเราอาจต้องการเปลี่ยนระดับการบันทึกโดยไม่ต้องสร้างภาพใหม่

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wsproxy-config
  labels:
    app: wsproxy
data:
  LOG_LEVEL: "debug"
  TARGET_HOST: "backend-service"
  TARGET_PORT: "8080"
```

จากนั้นใน Deployment เราจะอ้างอิง ConfigMap ผ่าน `envFrom`:
```yaml
envFrom:
  - configMapRef:
      name: wsproxy-config
```

วิธีนี้ทำให้สามารถปรับเปลี่ยนการตั้งค่าได้อย่างรวดเร็วโดยการอัปเดต ConfigMap และทำการ rolling update รีพอร์ติกโดยไม่ต้องสร้างภาพคอนเทนเนอร์ใหม่

## CI/CD

เพื่อให้มั่นใจว่าการเปลี่ยนแปลงใดๆ ที่ส่งไปยังคลังโค้ดจะผ่านการทดสอบและสามารถจัดส่งได้อย่างน่าเชื่อถือ เราตั้งค่าประบบ CI/CD โดยใช้ GitHub Actions ซึ่งช่วยให้เราอัตโนมัติกระบวนการสร้าง ทดสอบ ตรวจสอบ และจัดส่งภาพ Docker

### ci.yml - Continuous Integration

ไฟล์ `.github/workflows/ci.yml` กำหนดขั้นตอนการทำงานที่จะรันเมื่อมีการ push หรือ pull request ไปยังสาขาหลักหรือสาขาที่กำหนด งานนี้มุ่งเน้นไปที่การทดสอบ การตรวจสอบโค้ด และการตรวจสอบความปลอดภัยพื้นฐาน

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: ติดตั้ง Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: stable
      
      - name: ตรวจสอบรูปแบบโค้ด
        run: cargo fmt -- --check
      
      - name: รัน Clippy
        run: cargo clippy -- -D warnings
      
      - name: รันการทดสอบ
        run: cargo test --release
      
      - name: สร้างไบนารี
        run: cargo build --release
```

คำอธิบายขั้นตอน:
1. `actions/checkout@v3` ดึงโค้ดจากคลังมาทำงาน
2. ติดตั้ง Rust toolchain เวอร์ชันเสถียรล่าสุดโดยใช้ `dtolnay/rust-toolchain`
3. ตรวจสอบรูปแบบโค้ดด้วย `cargo fmt -- --check` เพื่อให้มั่นใจว่าโค้ดทั้งหมดปฏิบัติตามรูปแบบมาตรฐาน
4. รัน Clippy ซึ่งเป็นเครื่องมือตรวจสอบโค้ดของ Rust เพื่อหาข้อผิดพลาดทั่วไปและปรับปรุงโค้ด โดยเราให้มันทำการตรวจสอบแบบเข้มงวด (`-D warnings`) เพื่อให้การทำงานล้มเหลวหากมีการเตือนใดๆ
5. รันการทดสอบทั้งหมดในโหมดปล่อยด้วย `cargo test --release`
6. สร้างไบนารีในโหมดปล่อยด้วย `cargo build --release` เพื่อตรวจสอบว่าโค้ดสามารถสร้างได้สำเร็จ

หากขั้นตอนใดขั้นตอนหนึ่งล้มเหลว งานทั้งหมดจะถูกทำเครื่องหมายว่าล้มเหลวและการ merge pull request จะถูกบล็อกจนกว่าปัญหาจะได้รับการแก้ไข

### cd.yml - Continuous Deployment

ไฟล์ `.github/workflows/cd.yml` กำหนดขั้นตอนการทำงานที่จะรันเมื่อมีการ release ใหม่ถูกเผยแพร่ (หรือตามเงื่อนไขอื่นๆ เช่นการ push ไปยังสาขา release) งานนี้มุ่งเน้นไปที่การสร้างภาพ Docker การทดสอบภาพ และการส่งภาพไปยัง registry

```yaml
name: CD

on:
  release:
    types: [published]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: ตั้งค่า QEMU
        uses: docker/setup-qemu-action@v2
      
      - name: ตั้งค่า Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: เข้าสู่ระบบ Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      
      - name: สร้างและส่งภาพ Docker
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: bouroo/rs-wsproxy:latest
```

คำอธิบายขั้นตอน:
1. เช่นเคย เริ่มต้นด้วยการดึงโค้ดจากคลัง
2. ตั้งค่า QEMU ซึ่งช่วยให้เราสร้างภาพสำหรับหลายแพลตฟอร์มได้ (เช่น linux/amd64, linux/arm64) แม้จะรันบนผู้รันที่เป็น x86_64 ก็ตาม
3. ตั้งค่า Docker Buildx ซึ่งเป็นปลั๊กอิน CLI ของ Docker ที่ให้ความสามารถในการสร้างภาพขั้นสูงรวมถึงการสร้างหลายแพลตฟอร์ม
4. เข้าสู่ระบบ Docker Hub โดยใช้ความลับที่เก็บไว้ในคลัง (`secrets.DOCKERHUB_USERNAME` และ `secrets.DOCKERHUB_TOKEN`)
5. ใช้ `docker/build-push-action` เพื่อสร้างภาพจาก Dockerfile ในไดเรกทอรีปัจจุบันและส่งไปยัง Docker Hub ด้วยแท็ก `bouroo/rs-wsproxy:latest`

ด้วยการตั้งค่า CI/CD นี้ ทุกครั้งที่มีการเผยแพร่ release ใหม่ ภาพ Docker ล่าสุดจะถูกสร้างทดสอบและส่งไปยัง registry อัตโนมัติ ทำให้สามารถนำไปใช้งานในสภาพแวดล้อมต่างๆ ได้อย่างรวดเร็วและเชื่อถือได้

## สรุปทั้งซีรีส์

เราได้เดินทางมาถึงบทสุดท้ายของซีรีส์การสร้าง wsProxy ด้วย Rust แล้ว ในแปดบทที่ผ่านมาเราได้ครอบคลุมตั้งแต่พื้นฐานของ Rust จนถึงการสร้างผลิตภัณฑ์ที่พร้อมสำหรับการใช้งานในสภาพแวดล้อมผลิตจริง มาทบทวนสิ่งที่เราได้เรียนรู้กัน:

**บทที่ 1: พื้นฐาน Rust และการตั้งค่าโปรเจกต์**
- เรียนรู้ไวยากรณ์พื้นฐานของ Rust เช่นตัวแปร ประเภทข้อมูล และการควบคุมการไหล
- ตั้งค่าโปรเจกต์ Rust ใหม่ด้วย Cargo และเข้าใจโครงสร้างไดเรกทอรีมาตรฐาน
- เขียนโปรแกรม "Hello, World!" แรกใน Rust และรันมัน

**บทที่ 2: การทำงานกับไฟล์และการจัดการข้อผิดพลาด**
- สำรวจไลบรารีมาตรฐานของ Rust สำหรับการอ่านและเขียนไฟล์
- เข้าใจโมเดลการจัดการข้อผิดพลาดของ Rust ด้วย `Result` และ `Option`
- ฝึกเขียนฟังก์ชันที่คืนค่า `Result` และใช้ตัวดำเนินการ `?` เพื่อเผยแพร่ข้อผิดพลาด

**บทที่ 3: การเขียนโปรแกรมแบบไม่ซิงโครนัสด้วย Tokio**
- ทำความเข้าใจแนวคิดของการเขียนโปรแกรมแบบไม่ซิงโครนัสและเหตุผลที่มันสำคัญสำหรับแอปพลิเคชันเครือข่าย
- เรียนรู้วิธีการใช้งาน Tokio รันไทม์เพื่อจัดการงานแบบไม่ซิงโครนัส
- สำรวจแนวคิดของฟิวเจอร์และงาน (task) ใน Tokio

**บทที่ 4: WebSocket กับ Tokio-Tungstenite**
- ศึกษาโปรโตคอล WebSocket และการทำงานของมัน
- เรียนรู้การใช้ไลบรารี `tokio-tungstenite` เพื่อจัดการการเชื่อมต่อ WebSocket
- สร้างเซิร์ฟเวอร์ WebSocket พื้นฐานที่สามารถรับและส่งข้อความได้

**บทที่ 5: การออกแบบ CLI และการตั้งค่า**
- สำรวจไลบรารี `clap` สำหรับการสร้างอินเทอร์เฟซบรรทัดคำสั่งที่ทรงพลัง
- ออกแบบโครงสร้างการตั้งค่าแอปพลิเคชันโดยใช้ไลบรารี `config`
- นำไปใช้งานจริงด้วยการสร้าง CLI ที่สามารถรับอาร์กิวเมนต์และไฟล์การตั้งค่าได้

**บทที่ 6: การสร้างเซิร์ฟเวอร์ HTTP ด้วย Axum**
- ทำความรู้จักกับเฟรมเวิร์ก Axum ซึ่งสร้างอยู่บนพื้นฐานของ Tokio และ Tower
- สร้างเส้นทาง (route) และจัดการตัวประมวลผล (handler) สำหรับคำขอ HTTP ต่างๆ
- ผสานรวมการจัดการ WebSocket เข้ากับเซิร์ฟเวอร์ HTTP ของ Axum

**บทที่ 7: การเชื่อมต่อ TCP พื้นฐานและการจัดการข้อมูล**
- ศึกษาการสร้างการเชื่อมต่อ TCP ด้วย `tokio::net::TcpStream`
- เรียนรู้เทคนิคการอ่านและเขียนข้อมูลจากสตรีม TCP อย่างมีประสิทธิภาพ
- สำรวจการใช้ `BytesMut` เพื่อจัดการ buffer แบบไดนามิกเพื่อลดการจัดสรรหน่วยความจำ

**บทที่ 8: หัวใจของ Proxy, TLS, การทดสอบ และการจัดส่ง (บทปัจจุบัน)**
- สร้างฟังก์ชัน `connect_tcp` ที่ทนทานสำหรับการทำ DNS lookup และเชื่อมต่อ TCP พร้อมการตั้งค่า TCP_NODELAY
- ออกแบบและนำไปใช้งาน bidirectional pump ด้วย `tokio::select!` เพื่อจัดการการไหลของข้อมูลสองทิศทางพร้อมกัน
- อธิบายเหตุผลที่ต้องแยก TCP stream เป็น ReadHalf และ WriteHalf เพื่อหลีกเลี่ยง BiLock overhead
- นำไปใช้งาน TLS ด้วยไลบรารี `rustls` เพื่อความปลอดภัยของการเชื่อมต่อ
- อธิบายกลไกการปิดการทำงานอย่างสง่างาม (graceful shutdown) เพื่อจัดการสัญญาณระบบอย่างเหมาะสม
- สร้างการทดสอบหน่วยและการทดสอบแบบ integration เพื่อให้มั่นใจในความถูกต้องของโค้ด
- แสดงวิธีการจัดส่งด้วย Docker ผ่าน multi-stage build และ docker-compose สำหรับการพัฒนา
- อธิบายการจัดส่งใน Kubernetes ด้วย Deployment, Service และ ConfigMap
- ตั้งค่าระบบ CI/CD ด้วย GitHub Actions เพื่ออัตโนมัติกระบวนการทดสอบ สร้าง และจัดส่ง

ตลอดซีรีส์นี้เราได้เห็นว่า Rust ไม่ใช่แค่ภาษาที่ปลอดภัยและเร็วเท่านั้น แต่ยังให้เครื่องมือและไลบรารีที่ทรงพลังสำหรับการสร้างระบบที่ซับซ้อนเช่น WebSocket proxy ได้อย่างสะดวกและมั่นใจได้ ความปลอดภัยของหน่วยความภาพโดยไม่ต้องพึ่ง garbage collector ระบบประเภทที่เข้มงวด และคุณสมบัติเช่น pattern matching ทำให้การเขียนโปรโตคอลเครือข่ายเป็นไปอย่างถูกต้องและมีประสิทธิภาพ

ย้อนกลับไปตอนที่เห็นโพสต์จาก [rayrag.com](https://rayrag.com/) บน Facebook ที่เล่น RO บนเว็บเบราว์เซอร์ได้ ผมไม่นึกเลยว่าจะตื่นเต้นจนอยากสร้างของตัวเอง และได้เรียนรู้ Rust ทั้งหมดตั้งแต่ต้นจนจบซีรีส์นี้ จากความสงสัยง่ายๆ กลายเป็นโปรเจกต์จริงที่ใช้งานได้ นี่คือเสน่ห์ของการเขียนโปรแกรม — ความสงสัยนำไปสู่การเรียนรู้ และการเรียนรู้นำไปสู่การสร้างสรรค์

เราหวังว่าซีรีส์นี้จะเป็นประโยชน์ต่อผู้อ่านไม่ว่าจะเป็นผู้ที่เพิ่งเริ่มเรียนรู้ Rust หรือผู้ที่มีประสบการณ์แล้วแต่ต้องการดูตัวอย่างการสร้างแอปพลิเคชันจริงที่ใช้เทคนิคสมัยใหม่ของ Rust อย่างเต็มที่

ซอร์สโค้ดทั้งหมดของโครงการนี้สามารถดูได้ที่: https://github.com/bouroo/rs-wsProxy

← ก่อนหน้า: [สร้าง wsProxy — CLI, Config และ Server](/posts/rust/rust-wsproxy-server/)

ขอขอบคุณที่ติดตามอ่านจนจบซีรีส์นี้ หากคุณมีคำถาม ข้อเสนอแนะ หรือต้องการแบ่งปันประสบการณ์ในการสร้างแอปพลิเคชันด้วย Rust อย่าลังเลที่จะแสดงความคิดเห็นหรือติดต่อผู้เขียนผ่านช่องทางต่างๆ ที่ให้ไว้

Happy coding! 🚀