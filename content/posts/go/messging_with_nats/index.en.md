---
title: "Simple Messaging with Go + NATS"
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

featuredImage: "featured-image.jpg"
featuredImagePreview: "featured-image.jpg"

lightgallery: true
---

While waiting for a friend at a coffee shop, I had some free time and it occurred to me that we travel with asynchronous tasks. Whoever leaves first, does their work first, and arrives first. In the meantime, others are on their own journey, and eventually, we all meet at the destination. So, I decided to write a simple Go + NATS Pub/Sub to document how it's done.
<!--more-->

# Prerequisites

## Go
Since we're writing in Go, you can install it from the official Go website: [Download and install](https://go.dev/doc/install)

## Docker
For ease of use with NATS, let's also install Docker.
```bash
curl https://get.docker.com | bash
```

## NATS
Next up is NATS, which will be the intermediary for our messaging. We'll open a channel on `port 4222` by default, and also `8222` in case you want to send API requests for monitoring or other purposes. Here's the `compose.yaml` file:
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

# Let's start coding

## Publisher
Let's start with the job creator, the `publisher.go`. It will publish announcements on the `messages` topic every 5 seconds using a time Ticker until it's told to stop.
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
                msg := fmt.Sprintf("Publishing %d", time.Now().Unix())
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
The slave worker, `subscriber.go`, waits to receive jobs by subscribing to the `messages` topic until it's told to stop.

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
    sig := <-closeChan
    fmt.Printf("Received signal: %s. Exiting...\n", sig)
    return // Exit the loop and terminate the program
}
```

# Putting it all together

## Start NATS
```bash
docker compose up -d
```

## Start Publisher
```bash
go run publisher.go
```

## Start Subscriber
```bash
go run subscriber.go
```

###
Now, just watch the console for the results every 5 seconds until you terminate the process.
