---
title: "DragonflyDB vs Redis"
subtitle: ""
date: 2023-08-22T08:35:52+07:00
lastmod: 2023-08-22T08:35:53+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "เปรียบเทียบประสิทธิภาพระหว่าง DragonflyDB และ Redis 7 ด้วย memtier_benchmark ผ่าน TCP และ Unix socket พร้อมข้อมูลลิขสิทธิ์ 2024-2025"
license: ""
images: []
featuredImage: "featured-image.webp"
featuredImagePreview: "featured-image.webp"

tags: ["DevOps", "Database", "DragonflyDB", "Redis"]
categories: ["DevOps"]

lightgallery: true
---

ปกติแล้วเวลาเราใช้งาน In-memory database ก็มักจะนึกถึง Redis กับ Memcache วันนี้เลยจะมาแนะนำอีกตัวนึงที่น่าสนใจนั่นคือ Dragonfly ซึ่งเคลมตัวเองว่าเป็น **multi-threaded Redis replacement** เลยมาลองทดสอบดูว่าระหว่าง Dragonfly กับ Redis 7 ผลจะเป็นอย่างไร

<!--more-->

## Redis test compose

เริ่มจากสร้าง compose สำหรับทดสอบ Redis

```yaml
services:
  redis:
    image: redis
    ulimits:
      memlock: -1
    volumes:
      - redis-data:/data
    hostname: redis
    command:
      ["redis-server", "--loglevel warning", "--unixsocket", "/data/redis.sock"]

  redis_tcp:
    depends_on:
      - redis
    image: redislabs/memtier_benchmark
    volumes:
      - redis-data:/redis-data
      - ./tmp:/data
    command: ["-h", "redis", "--out-file=/data/redis-tcp.log"]

  redis_sock:
    depends_on:
      - redis
    image: redislabs/memtier_benchmark
    volumes:
      - redis-data:/redis-data
      - ./tmp:/data
    command: ["-S", "/redis-data/redis.sock", "--out-file=/data/redis-sock.log"]

volumes:
  redis-data:
```

## Dragonfly test compose

ต่อด้วย compose สำหรับ Dragonfly

{{< admonition note "Note" >}}
ปกติ Dragonfly จะใช้ Ram Limit ขั้นต่ำต่อ Logical CPU อยู่ที่ 256MB นะครับ
{{< /admonition >}}

```yaml
services:
  dragonfly:
    image: docker.dragonflydb.io/dragonflydb/dragonfly
    ulimits:
      memlock: -1
    volumes:
      - dragonfly-data:/data
    hostname: dragonfly
    command:
      ["dragonfly", "--logtostderr", "--unixsocket", "/data/dragonfly.sock"]

  dragonfly_tcp:
    depends_on:
      - dragonfly
    image: redislabs/memtier_benchmark
    volumes:
      - dragonfly-data:/dragonfly-data
      - ./tmp:/data
    command: ["-h", "dragonfly", "--out-file=/data/dragonfly-tcp.log"]

  dragonfly_sock:
    depends_on:
      - dragonfly
    image: redislabs/memtier_benchmark
    volumes:
      - dragonfly-data:/dragonfly-data
      - ./tmp:/data
    command: ["-S", "/dragonfly-data/dragonfly.sock", "--out-file=/data/dragonfly-sock.log"]

volumes:
  dragonfly-data:
```
## TCP

เริ่มจากท่าที่ทุกคนน่าจะใช้งานกันปกติทดสอบผ่านทาง TCP/IP ระหว่าง container

### Redis

```bash
docker compose -f docker-compose.redis.yml up redis_tcp 
```

| Type      | Ops/sec    | Hits/sec   | Misses/sec    | Avg. Latency | p50 Latency | p99 Latency | p99.9 Latency | KB/sec   |
|-----------|------------|------------|---------------|--------------|-------------|-------------|--------------|----------|
| Sets      | 6446.59    | ---        | ---           | 2.82053      | 2.68700     | 5.40700     | 10.36700     | 496.50   |
| Gets      | 64395.05   | 0.00       | 64395.05      | 2.82098      | 2.68700     | 5.40700     | 10.68700     | 2508.47  |
| Waits     | 0.00       | ---        | ---           | ---          | ---         | ---         | ---          | ---      |
| Totals    | 70841.64   | 0.00       | 64395.05      | 2.82093      | 2.68700     | 5.40700     | 10.62300     | 3004.97  |

### Dragonfly

```bash
docker compose -f docker-compose.dragonfly.yml up dragonfly_tcp 
```

| Type      | Ops/sec    | Hits/sec   | Misses/sec    | Avg. Latency | p50 Latency | p99 Latency | p99.9 Latency | KB/sec   |
|-----------|------------|------------|---------------|--------------|-------------|-------------|--------------|----------|
| Sets      | 12683.03   | ---        | ---           | 1.53450      | 0.66300     | 10.87900    | 16.12700     | 976.82   |
| Gets      | 126690.88  | 0.00       | 126690.88     | 1.51768      | 0.65500     | 10.94300    | 16.31900     | 4935.16  |
| Waits     | 0.00       | ---        | ---           | ---          | ---         | ---         | ---          | ---      |
| Totals    | 139373.91  | 0.00       | 126690.88     | 1.51921      | 0.65500     | 10.94300    | 16.31900     | 5911.97  |

## UNIX socket

แล้วก็มาลองทดสอบด้วยการใช้ UNIX socket ระหว่าง container ดู

### Redis

```bash
docker compose -f docker-compose.redis.yml up redis_sock 
```

| Type      | Ops/sec    | Hits/sec   | Misses/sec    | Avg. Latency | p50 Latency | p99 Latency | p99.9 Latency | KB/sec   |
|-----------|------------|------------|---------------|--------------|-------------|-------------|--------------|----------|
| Sets      | 17540.93   | ---        | ---           | 1.04733      | 0.99900     | 1.82300     | 4.99100      | 1350.96  |
| Gets      | 175216.52  | 0.00       | 175216.52     | 1.04597      | 0.99900     | 1.82300     | 3.87100      | 6825.44  |
| Waits     | 0.00       | ---        | ---           | ---          | ---         | ---         | ---          | ---      |
| Totals    | 192757.45  | 0.00       | 175216.52     | 1.04610      | 0.99900     | 1.82300     | 4.03100      | 8176.40  |

### Dragonfly

```bash
docker compose -f docker-compose.dragonfly.yml up dragonfly_sock 
```

| Type      | Ops/sec    | Hits/sec   | Misses/sec    | Avg. Latency | p50 Latency | p99 Latency | p99.9 Latency | KB/sec   |
|-----------|------------|------------|---------------|--------------|-------------|-------------|--------------|----------|
| Sets      | 23136.83   | ---        | ---           | 0.80438      | 0.27900     | 7.74300     | 14.39900     | 1781.94  |
| Gets      | 231114.05  | 0.00       | 231114.05     | 0.78618      | 0.27900     | 7.39100     | 12.67100     | 9002.89  |
| Waits     | 0.00       | ---        | ---           | ---          | ---         | ---         | ---          | ---      |
| Totals    | 254250.88  | 0.00       | 231114.05     | 0.78783      | 0.27900     | 7.42300     | 12.79900     | 10784.83 |

## ภาพรวมลิขสิทธิ์ (2024-2025)

วงการ in-memory database มีการเปลี่ยนแปลงลิขสิทธิ์ครั้งใหญ่หลังจากที่เขียน benchmark นี้:

- **Redis** — ในเดือนมีนาคม 2024 Redis Ltd. เปลี่ยนจาก BSD เป็น dual license SSPL/RSALv2 และใน Redis 8.0 (2025) เพิ่ม AGPLv3 เป็นตัวเลือกที่สาม การเปลี่ยนแปลงนี้เป็นสาเหตุที่ทำให้เกิด community fork ด้านล่าง
- **Valkey** — เป็น fork ของ Redis 7.2.4 ภายใต้ลิขสิทธิ์ BSD สร้างโดย Linux Foundation (มีนาคม 2024) โดยมี AWS, Google Cloud และ Oracle สนับสนุน Valkey 8.1 (มีนาคม 2025) มี throughput สูงขึ้นประมาณ 8%, P99 latency ต่ำลง 22% และใช้ memory น้อยลง 20% เมื่อเทียบกับ Redis OSS
- **Dragonfly** — ใช้ Business Source License (BSL 1.1) มาตั้งแต่แรก BSL เป็น source-available และใช้งานได้ฟรีสำหรับ self-hosting โดยจะเปลี่ยนเป็น Apache 2.0 หลังจาก change date (ปกติประมาณสี่ปี) Dragonfly ไม่ใช่ open source ที่ผ่านการรับรองโดย OSI แต่ไม่มีข้อจำกัดเหมือน SSPL ของ Redis สำหรับผู้ให้บริการ managed service

ถ้าลิขสิทธิ์สำคัญสำหรับการใช้งานของคุณ (เช่น ต้องการให้บริการ managed service) Valkey (BSD) จะเป็นตัวเลือกที่เปิดกว้างที่สุด BSL ของ Dragonfly มีข้อจำกัดมากกว่า BSD แต่ไม่ซับซ้อนเท่า SSPL ของ Redis สำหรับ self-hosted benchmark ทั้งสามตัวใช้งานได้ฟรี

## สรุป
ด้วยพลังแห่ง Multi-thread ทำให้ผลที่ออกมา Dragonfly ดูดีเลยทีเดียว (ยิ่งผ่าน UNIX socket ยิ่งเห็นความต่างจาก TCP/IP) แต่ก็มีข้อสังเกตุคือ ตอนนี้ Dragonfly ไม่ได้รองรับ Redis API แบบ 100% นะครับ ซึ่ง ณ ปี 2025 ความเข้ากันได้ของ Redis API บน Dragonfly ดีขึ้นอย่างมาก แต่บางคำสั่ง edge-case อาจยังมีพฤติกรรมต่างกัน — ควรเช็ค [compatibility reference](https://www.dragonflydb.io/docs/command-reference/compatibility) สำหรับ use case ของคุณเสมอ ถ้ามีการใช้ท่ายากท่าพิเศษ จะต้องเช็คกันก่อนว่าสามารถเอามาใช้งานแทนได้เลยหรือไม่ ที่ [command-reference](https://www.dragonflydb.io/docs/command-reference/compatibility) ซึ่งจากการทดลองใช้งานดูแล้ว ถ้าโดยปกติใช้งาน Redis แค่ Instance เดียว ไม่ได้ใช้งาน ด้าน `Graph`, `Geo location` โดยใช้แค่ หลัก ๆ เป็น `Set` `Get` และใช้ `PubSub` `Stream` นิดหน่อย ก็สามารถแทนด้วย Dragonfly ได้เลย โดยโค้ดและการเชื่อมต่อยังใช้ตามเดิม
