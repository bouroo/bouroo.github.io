---
title: "Messaging อย่างง่าย Go + NATS"
subtitle: ""
date: 2024-09-28T10:55:17+07:00
lastmod: 2024-09-28T10:55:17+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: ""
license: ""
images: []

tags: []
categories: []

featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"

lightgallery: true
---

ระหว่างที่รอเพื่อนร่วมทางที่ร้านกาแฟ พอมีเวลาว่างเลยนึกขึ้นได้ว่าเราเดินทางกับแบบ asynchronous task เลยนี่หว่า ใครออกมาก่อนทำก่อนถึงก่อน ในระหว่างที่คนอื่นก็เดินทางของตัวเองไป แล้วสุดท้ายมาเจอกันปลายทาง ก็เลยลองเขียน Go + NATS ทำ Pub/Sub ง่อย ๆ บันทึกไว้ว่ามันทำยังไง
<!--more-->

# ของที่ต้องมีก่อนจะเริ่ม

## Go
ก็เขียนด้วย Go ละนะ Install ได้ตาม official ของ Go ได้เลยจ้า [Download and install](https://go.dev/doc/install)

## Docker
เพื่อความง่ายในการใช้ NATS ก็ลง Docker ไว้กันด้วยนะ
```bash
curl https://get.docker.com | bash
```

## NATS
ต่อมาที่ NATS ที่จะใช้เป็นตัวกลางของ Messaging ของเราโดยจะเปิดช่องทำงานผ่าน `port 4222` by default ละก็ 8222 เปิดไว้เผื่ออยากยิง API เข้าไปเช็ค monitoring อะไรก็ว่ากันไป ด้วยไฟล์ `compose.yaml`
```yaml
services:
    nats:
        image: nats
        container_name: nats-server
        command: ["--http_port", "8222"]
        networks:
            - nats-net
        ports:
            - 4222:4222
            - 8222:8222

networks:
    nats-net:
        external: false
        name: nats-net
```

# เริ่ม coding

## Publisher
เริ่มจากคนสร้างงานสร้างอาชีพก่อนเลยคือ `publisher.go` เป็นคนลงประกาศ Publish ผ่านทางหัวข้อ `messages` ทุก ๆ 5 วินาทีด้วย time Ticker เรื่อย ๆ จนกว่าได้โดนสั่งให้หยุด
```go
package main

import (
    "fmt"
	"os"
	"os/signal"
	"syscall"
    "log"
    "time"

    "github.com/nats-io/nats.go"
)

func main() {
    // Create a channel to listen for OS signals
	closeChan := make(chan os.Signal, 1)

	// Notify the channel when SIGINT (Ctrl+C) or SIGTERM is received
	signal.Notify(closeChan, os.Interrupt, syscall.SIGTERM)

	// Create a ticker that ticks every 5 seconds
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

    // Connect to NATS server
    nc, err := nats.Connect(nats.DefaultURL)
    if err != nil {
        log.Fatalf("Error connecting to NATS server: %v", err)
    }
    defer nc.Close()

	fmt.Println("Press Ctrl+C to exit...")

    // Publish messages every 5 seconds until terminate signal
    for {
        select {
            case <-ticker.C:
                // Publish messages every 5 seconds
                msg := fmt.Sprintf("Publishing %d", i)
                err := nc.Publish("messages", []byte(msg))
                if err != nil {
                    log.Fatalf("Error publishing message: %v", err)
                }
                fmt.Printf("Published: %s\n", msg)
            case sig := <-closeChan:
                // This block executes when a termination signal is received
                fmt.Printf("Received signal: %s. Exiting...\n", sig)
                return // Exit the loop and terminate the program
		}
    }
}
```

## Subscriber
แรงงานทาส `subscriber.go` ที่รอรับงานผ่านการ subscribe หัวข้อ `messages` เรื่อย ๆ จนกว่าได้โดนสั่งให้หยุด

```go
package main

import (
    "fmt"
	"os"
	"os/signal"
	"syscall"
    "log"

    "github.com/nats-io/nats.go"
)

func main() {
    // Create a channel to listen for OS signals
	closeChan := make(chan os.Signal, 1)

	// Notify the channel when SIGINT (Ctrl+C) or SIGTERM is received
	signal.Notify(closeChan, os.Interrupt, syscall.SIGTERM)

    // Connect to NATS server
    nc, err := nats.Connect(nats.DefaultURL)
    if err != nil {
        log.Fatalf("Error connecting to NATS server: %v", err)
    }
    defer nc.Close()

    // Subscribe to the "messages" subject
    _, err = nc.Subscribe("messages", func(msg *nats.Msg) {
        fmt.Printf("Received message: %s\n", string(msg.Data))
    })
    if err != nil {
        log.Fatalf("Error subscribing to subject: %v", err)
    }

    // This block executes when a termination signal is received
    sig := <-closeChan:
    fmt.Printf("Received signal: %s. Exiting...\n", sig)
    return // Exit the loop and terminate the program
}
```

# รวมร่าง

## Start NATS
```bash
docker compose up -d
```

## Start Pubplisher
```bash
go run publisher.go
```

## Start Subscriber
```bash
go run subscriber.go
```

###
จบรอดูผลงานผ่าน console ทุก ๆ 5 วินาทีจนกว่าจะสั่ง terminate