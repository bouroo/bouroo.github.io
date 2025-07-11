---
title: "เลือก Messaging System ตัวไหนดี? Kafka, Valkey, RabbitMQ, NATS... โอย เยอะไปหมด!"
date: 2025-07-11T10:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
tags: ["messaging", "kafka", "valkey", "redis", "rabbitmq", "nats"]
categories: ["devops", "programming"]
resources:
- name: "featured-image"
  src: "featured-image.jpg"
featuredImage: "featured-image.jpg"
featuredImagePreview: "featured-image"
---

เวลาจะทำระบบที่มันคุยกันเยอะๆ (distributed systems) เนี่ย เรื่องปวดหัวอย่างนึงคือจะเลือกระบบ Messaging ตัวไหนดี มันมีให้เลือกเยอะซะเหลือเกิน ทั้ง Kafka, Valkey (ที่เกิดใหม่จาก Redis), RabbitMQ, แล้วก็ NATS อีก แต่ละตัวก็มีดีมีเสียต่างกันไป งั้นโพสต์นี้ขอมาจดโน้ตเทียบให้ตัวเองดูหน่อยละกัน ว่าตัวไหนมันเหมาะกับงานแบบไหน

<!--more-->

## Kafka

### มันคืออะไร?

Kafka นี่มันคือแพลตฟอร์มสตรีมมิ่งข้อมูลขนาดใหญ่ยักษ์เลยนะ ออกแบบมาเพื่อรองรับข้อมูลแบบเรียลไทม์ที่ไหลเข้ามามหาศาล คิดภาพเหมือนท่อส่งข้อมูลขนาดใหญ่ที่ทั้งเร็ว ทนทาน แล้วก็ขยายระบบได้ง่าย

{{< mermaid >}}
graph LR
    Producer --> Kafka[Kafka Broker]
    Kafka --> Consumer
    subgraph Kafka Broker
        Topic1[Topic 1]
        Topic2[Topic 2]
    end
    Producer -- Writes to --> Topic1
    Consumer -- Reads from --> Topic1
{{< /mermaid >}}

### แล้วจะใช้ตอนไหนดี?

*   **อยากวิเคราะห์ข้อมูลสดๆ:** เวลาที่ต้องการประมวลผลข้อมูลที่ไหลเข้ามาเรื่อยๆ เพื่อให้ได้ผลลัพธ์ทันที
*   **เก็บ Log จากทุกที่:** รวบรวม Log จากหลายๆ เซอร์วิสมาไว้ที่เดียวกัน
*   **ทำ Event Sourcing:** ใช้เก็บทุกเหตุการณ์ที่เกิดขึ้นในระบบเป็นแหล่งข้อมูลหลัก

### ที่ชอบ

*   รับส่งข้อมูลได้เยอะและเร็วมาก (High throughput, low latency)
*   ขยายระบบง่าย ไม่ต้องกลัวล่ม (Scalable, fault-tolerant)
*   เก็บข้อความได้นาน ไม่หาย (Durable message storage)

### ที่ไม่ค่อยชอบ

*   ติดตั้งกับดูแลรักษายุ่งยากพอตัว
*   กินทรัพยากรเครื่องเยอะกว่าตัวอื่น
*   ไม่ค่อยเหมาะกับงานคิวแบบง่ายๆ ที่อยากให้ worker มาหยิบงานไปทำทีละชิ้นแล้วจบ

## Valkey (Redis)

### มันคืออะไร?

Valkey หรือที่เรารู้จักกันในนาม Redis นั่นแหละ (Valkey เป็น fork ที่ community ช่วยกันดูแล) มันเป็นฐานข้อมูลในหน่วยความจำที่เร็วปรื๊ด คนเลยนิยมเอามาทำเป็น cache หรือ message broker แบบง่ายๆ โดยใช้ data structure อย่าง list มาทำเป็นคิว

{{< mermaid >}}
graph LR
    Producer --> Redis[Valkey (Redis)]
    Redis --> Consumer
    subgraph Valkey (Redis)
        List[List (Queue)] 
        PubSub[Pub/Sub Channel]
    end
    Producer -- LPUSH --> List
    Consumer -- BRPOP --> List
    Publisher -- PUBLISH --> PubSub
    Subscriber -- SUBSCRIBE --> PubSub
{{< /mermaid >}}

### แล้วจะใช้ตอนไหนดี?

*   **ทำคิวแบบบ้านๆ:** งานที่ไม่ต้องการความสามารถสูงส่งอะไรมาก แค่คิวธรรมดาๆ ก็พอ
*   **ทำ Pub/Sub:** เหมาะกับงานแชท หรือส่ง notification แบบ real-time
*   **ทำ Cache:** อันนี้งานถนัดเค้าเลย

### ที่ชอบ

*   เร็วโคตรๆ เพราะทำงานบน RAM
*   ใช้ง่าย ติดตั้งง่าย ไม่ซับซ้อน
*   ทำได้หลายอย่างดี มี data structure ให้เล่นเยอะ

### ที่ไม่ค่อยชอบ

*   ถ้าไม่ตั้งค่าดีๆ ไฟดับทีข้อมูลหายเกลี้ยง (Not durable by default)
*   เก็บข้อความขนาดใหญ่มากไม่ได้
*   ไม่ได้เกิดมาเพื่องานสตรีมข้อมูลหนักๆ หรือการ routing ข้อความซับซ้อน

## RabbitMQ

### มันคืออะไร?

RabbitMQ นี่เป็น Message Broker รุ่นใหญ่ที่เก๋าเกมมาก ใช้โปรโตคอล AMQP เป็นหลัก ขึ้นชื่อเรื่องความสามารถในการส่งข้อความที่ครบเครื่อง, การ routing ที่ยืดหยุ่น และความน่าเชื่อถือ

{{< mermaid >}}
graph LR
    Producer --> Exchange[Exchange]
    Exchange --> Queue1[Queue 1]
    Exchange --> Queue2[Queue 2]
    Queue1 --> Consumer1
    Queue2 --> Consumer2
    subgraph RabbitMQ Broker
        Exchange
        Queue1
        Queue2
    end
    Producer -- Publishes to --> Exchange
    Exchange -- Routes to --> Queue1
    Exchange -- Routes to --> Queue2
    Consumer1 -- Consumes from --> Queue1
    Consumer2 -- Consumes from --> Queue2
{{< /mermaid >}}

### แล้วจะใช้ตอนไหนดี?

*   **ทำงานเบื้องหลัง (Async):** แยกงานหนักๆ ออกไปทำเบื้องหลัง ไม่ต้องให้ user รอ
*   **กระจายงาน (Work queues):** มีงานกองอยู่ อยากให้ worker หลายๆ ตัวมาช่วยกันทำ
*   **ส่งข้อความแบบมีเงื่อนไข:** อยากส่งข้อความไปหา consumer ที่เฉพาะเจาะจงตามเงื่อนไขต่างๆ

### ที่ชอบ

*   ฟีเจอร์เยอะมาก เป็นผู้ใหญ่ในวงการ
*   การ routing ข้อความยืดหยุ่นสุดๆ
*   รองรับรูปแบบการส่งข้อความได้หลากหลาย (fanout, direct, topic)

### ที่ไม่ค่อยชอบ

*   ถ้าเทียบกับ Kafka เรื่อง throughput อาจจะสู้ไม่ได้
*   ถ้าอยากให้มันทนทานต่อความผิดพลาด (high availability) ต้องตั้งค่าดีๆ
*   ปกติข้อความจะถูกอ่านแล้วลบทิ้ง ไม่ได้ออกแบบมาให้เก็บไว้นานๆ

## NATS

### มันคืออะไร?

NATS เป็นระบบ Messaging ที่เกิดมาเพื่อยุค Cloud-Native, IoT, และ Microservices เลย จุดเด่นคือความเร็วสูงและเบามาก เน้นความเรียบง่ายเป็นหลัก

{{< mermaid >}}
graph LR
    Producer --> NATS[NATS Server]
    NATS --> Consumer
    subgraph NATS Server
        TopicA[Topic A]
        TopicB[Topic B]
    end
    Producer -- Publishes to --> TopicA
    Consumer -- Subscribes to --> TopicA
{{< /mermaid >}}

### แล้วจะใช้ตอนไหนดี?

*   **ให้ Microservices คุยกัน:** สื่อสารระหว่างเซอร์วิสแบบเร็วๆ และไว้ใจได้
*   **ส่งข้อมูลจากอุปกรณ์ IoT:** เหมาะกับการรับข้อความเล็กๆ จำนวนมหาศาลจากอุปกรณ์ต่างๆ
*   **ระบบสั่งการ (Command and control):** ส่งคำสั่งไปควบคุมระบบที่กระจายอยู่ตามที่ต่างๆ

### ที่ชอบ

*   เร็วและเบามากจริงๆ
*   ติดตั้งและจัดการง่าย
*   รองรับทั้ง Pub/Sub และ Request/Reply

### ที่ไม่ค่อยชอบ

*   ตัวมันเองไม่มีระบบเก็บข้อความ (ต้องใช้ NATS Streaming/JetStream เข้ามาช่วย)
*   ฟีเจอร์การ routing ไม่ซับซ้อนเท่า RabbitMQ
*   ไม่ได้ออกแบบมาให้เก็บข้อความระยะยาวเหมือน Kafka

## สรุปแล้วจะเลือกตัวไหนดี?

สุดท้ายแล้ว การจะเลือกตัวไหนมันก็ขึ้นอยู่กับว่าเราจะเอามันไปทำอะไร:

*   **อยากได้ท่อส่งข้อมูลยักษ์ใหญ่ ทนทาน ขยายง่าย:** ไป **Kafka** เลย ไม่ผิดหวัง
*   **อยากได้คิวเร็วๆ ง่ายๆ หรือทำ Pub/Sub, Cache:** **Valkey (Redis)** คือคำตอบ
*   **อยากได้ Message Broker ที่เก๋าเกม routing ซับซ้อนได้ ไว้ใจได้:** ต้อง **RabbitMQ**
*   **อยากได้ระบบส่งข้อความที่เร็ว เบา เหมาะกับ Microservices:** **NATS** คือตัวเลือกที่น่าสนใจมาก

ก่อนจะตัดสินใจ ก็ลองถามตัวเองดูว่า งานของเราต้องการ throughput ขนาดไหน, ต้องเก็บข้อมูลนานแค่ไหน, การส่งข้อความซับซ้อนรึเปล่า, แล้วเรามีแรงดูแลมันมากน้อยแค่ไหน
