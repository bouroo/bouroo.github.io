---
title: "Async Rust with Tokio"
subtitle: ""
date: 2026-07-12T09:00:00+07:00
lastmod: 2026-07-12T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "async/await, the Tokio runtime, spawning tasks, channels, and async I/O for network applications"
license: ""
images: []
tags: ["Rust", "Tutorial", "Tokio"]
categories: ["Rust"]
featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"
lightgallery: true
---

<!--more-->

# Async Rust with Tokio

Asynchronous programming is essential for building high-performance network applications that can handle many concurrent connections without the overhead of spawning a thread for each connection. In this tutorial, we'll explore async/await syntax, the Tokio runtime, spawning tasks, channels for communication between tasks, asynchronous I/O for network programming, using `tokio::select!` for handling multiple futures, graceful shutdown mechanisms, and how these concepts apply to a real-world WebSocket proxy project (rs-wsProxy).

## Why Async?

Traditional network servers often use a thread-per-connection model, where each incoming connection gets its own thread. While simple to understand and implement, this approach doesn't scale well to thousands of concurrent connections because each thread consumes significant memory (typically 1-2MB stack space) and involves expensive context switching costs imposed by the operating system.

Asynchronous programming offers a more efficient alternative. Instead of blocking an entire thread while waiting for I/O operations (like reading from a socket), async code allows the runtime to suspend the current task and switch to another task that is ready to make progress. This enables a single thread to manage thousands of concurrent connections with minimal overhead.

However, Rust's async/await syntax alone doesn't provide a runtime to execute asynchronous code. Unlike languages with built-in runtimes (like Go or Node.js), Rust's async/await is a low-level feature that requires an external runtime to drive the asynchronous tasks to completion. This is where Tokio comes in – it provides a mature, feature-rich runtime for writing asynchronous applications in Rust.

## async/await

The `async` and `await` keywords are the foundation of asynchronous programming in Rust. When you mark a function with `async`, it returns a `Future` – a value that represents a computation that may not have completed yet. The `.await` keyword is used to wait for a future to complete, yielding control back to the runtime so other tasks can run while waiting.

Here's a basic example:

```rust
async fn hello_world() {
    println!("Hello, world!");
}

#[tokio::main]
async fn main() {
    hello_world().await;
}
```

In this example:
- `hello_world` is an asynchronous function that returns a `Future`
- When we call `hello_world().await`, we're telling the runtime to execute this future until it completes
- The `#[tokio::main]` attribute sets up the Tokio runtime and runs our async `main` function

You can also use `async` blocks for inline asynchronous code:

```rust
#[tokio::main]
async fn main() {
    let future = async {
        println!("Inside async block");
        42
    };
    
    let result = future.await;
    println!("Result: {}", result);
}
```

Futures are lazy – they don't do anything until you `.await` them. This allows you to combine multiple futures and execute them concurrently.

## Tokio Runtime

Tokio provides multiple runtime configurations to suit different workloads. The most common way to start a Tokio application is with the `#[tokio::main]` attribute macro:

```rust
#[tokio::main]
async fn main() {
    println!("Hello from Tokio!");
}
```

By default, this creates a multi-threaded runtime with worker threads equal to the number of CPU cores. You can customize the runtime using `tokio::runtime::Builder`:

```rust
use tokio::runtime::Builder;

fn main() {
    let runtime = Builder::new_multi_thread()
        .worker_threads(4) // Use 4 worker threads
        .enable_all()      // Enable all features (timers, I/O, etc.)
        .build()
        .unwrap();
    
    runtime.block_on(async {
        println!("Running on custom Tokio runtime");
    });
}
```

The runtime builder offers various configurations:
- `new_current_thread()`: Single-threaded runtime (good for debugging or when you want deterministic behavior)
- `new_multi_thread()`: Multi-threaded runtime (default for `#[tokio::main]`)
- `worker_threads(n)`: Set the number of worker threads
- `enable_io()`, `enable_time()`, etc.: Enable specific features
- `thread_name()`: Set custom thread names for easier debugging

For I/O-heavy applications like network proxies, the multi-threaded runtime typically provides the best performance.

## Spawning Tasks

While async functions allow you to write asynchronous code, you often need to run multiple concurrent tasks independently. Tokio provides `tokio::spawn` for this purpose:

```rust
use tokio::{self, time::{sleep, Duration}};

#[tokio::main]
async fn main() {
    // Spawn a new task
    let handle = tokio::spawn(async {
        println!("Task started");
        sleep(Duration::from_secs(2)).await;
        println!("Task completed");
        42
    });

    // Do other work while the task runs
    println!("Doing other work...");
    
    // Wait for the task to complete and get its result
    let result = handle.await.unwrap();
    println!("Task returned: {}", result);
}
```

Key points about `tokio::spawn`:
- It returns a `JoinHandle<T>` where `T` is the output type of the spawned future
- The `JoinHandle` allows you to await the task's completion and retrieve its result
- Spawned tasks run independently on the Tokio runtime
- Use the `move` keyword to capture variables from the surrounding scope:
  
```rust
let data = vec![1, 2, 3];
let handle = tokio::spawn(move || {
    // `data` is moved into the task
    println!("Data: {:?}", data);
});
```

## Channels

Tasks often need to communicate with each other. Tokio provides several channel types for asynchronous communication:

### Multi-Producer, Single-Consumer (mpsc)

Ideal for scenarios where multiple producers send messages to a single consumer:

```rust
use tokio::sync::mpsc;
use tokio::{self, time::{sleep, Duration}};

#[tokio::main]
async fn main() {
    // Create a channel with buffer capacity of 32
    let (tx, mut rx) = mpsc::channel(32);
    
    // Spawn three producer tasks
    for i in 0..3 {
        let tx = tx.clone(); // Clone the sender for each task
        tokio::spawn(async move {
            for j in 0..3 {
                let msg = format!("Task {} message {}", i, j);
                tx.send(msg).await.unwrap();
                sleep(Duration::from_millis(100)).await;
            }
        });
    }
    
    // Consumer task
    tokio::spawn(async move {
        while let Some(msg) = rx.recv().await {
            println!("Received: {}", msg);
        }
        println!("All messages received");
    });
    
    // Give time for messages to be processed
    sleep(Duration::from_secs(2)).await;
}
```

### One-Shot Channels

Useful for one-time communication between two tasks:

```rust
use tokio::sync::oneshot;
use tokio::{self, time::{sleep, Duration}};

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();
    
    // Spawn a task that will send a value
    tokio::spawn(async move {
        sleep(Duration::from_secs(1)).await;
        let _ = tx.send(42); // Ignore send error if receiver dropped
    });
    
    // Wait for the value
    match rx.await {
        Ok(value) => println!("Received: {}", value),
        Err(_) => println!("Sender dropped"),
    }
}
```

## Async I/O

Tokio provides asynchronous versions of standard I/O types like `TcpListener`, `TcpStream`, `UdpSocket`, etc. These types implement the `AsyncReadExt` and `AsyncWriteExt` traits from the `tokio::io` module, providing async versions of read/write operations.

Here's a simple async echo server:

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Listening on 127.0.0.1:8080");
    
    loop {
        let (socket, addr) = listener.accept().await?;
        println!("Accepted connection from: {}", addr);
        
        // Handle each connection in a separate task
        tokio::spawn(async move {
            if let Err(e) = process_socket(socket).await {
                eprintln!("Error processing socket: {}", e);
            }
        });
    }
}

async fn process_socket(mut socket: TcpStream) -> Result<(), Box<dyn std::error::Error>> {
    let mut buffer = [0; 1024];
    
    loop {
        let n = socket.read(&mut buffer).await?;
        if n == 0 {
            // Connection closed
            break;
        }
        
        // Echo the data back
        socket.write_all(&buffer[..n]).await?;
    }
    
    Ok(())
}
```

This server:
1. Binds to localhost:8080
2. Accepts incoming connections in a loop
3. Spawns a new task for each connection to handle it concurrently
4. Each task reads data from the socket and writes it back (echo)

## tokio::select!

The `tokio::select!` macro allows you to wait on multiple asynchronous operations simultaneously, proceeding as soon as one of them completes. This is particularly useful for implementing timeouts, handling multiple communication channels, or implementing select-like behavior for network I/O.

Here's a timeout example:

```rust
use tokio::{self, time::{sleep, Duration, timeout}};

#[tokio::main]
async fn main() {
    let future = async {
        sleep(Duration::from_secs(5)).await;
        println!("This takes 5 seconds");
    };
    
    // Wait for the future to complete, but timeout after 2 seconds
    match timeout(Duration::from_secs(2), future).await {
        Ok(_) => println!("Completed"),
        Err(_) => println!("Timed out!"),
    }
}
```

And here's an example using `select!` directly for more complex scenarios:

```rust
use tokio::{self, sync::mpsc, time::{sleep, Duration, interval}};

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32);
    let mut interval = interval(Duration::from_secs(2));
    
    // Spawn a sender that sends messages every second
    tokio::spawn(async move {
        let mut i = 0;
        loop {
            let _ = tx.send(i).await;
            i += 1;
            sleep(Duration::from_secs(1)).await;
        }
    });
    
    // Use select! to handle both incoming messages and periodic ticks
    loop {
        tokio::select! {
            msg = rx.recv() => {
                match msg {
                    Some(value) => println!("Received: {}", value),
                    None => {
                        println!("Sender disconnected");
                        break;
                    }
                }
            }
            _ = interval.tick() => {
                println!("Periodic tick");
            }
        }
                println!("Interval tick");
            }
        }
    }
}
```

In network applications like proxies, it's crucial to handle shutdown signals gracefully to avoid dropping connections abruptly. Tokio provides utilities for handling Unix signals like SIGINT (Ctrl+C) and SIGTERM.

Here's an example of graceful shutdown handling:

```rust
use tokio::{self, net::TcpListener, signal};
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Listening on 127.0.0.1:8080");
    
    // Create a future that resolves when shutdown signal is received
    let shutdown_signal = async {
        signal::ctrl_c().await
            .expect("Failed to install Ctrl+C handler");
    };
    
    // Main accept loop
    loop {
        tokio::select! {
            // Accept new connections
            res = listener.accept() => {
                match res {
                    Ok((socket, addr)) => {
                        println!("Accepted connection from: {}", addr);
                        tokio::spawn(async move {
                            if let Err(e) = handle_connection(socket).await {
                                eprintln!("Connection error: {}", e);
                            }
                        });
                    }
                    Err(e) => {
                        eprintln!("Accept error: {}", e);
                    }
                }
            }
            // Wait for shutdown signal
            _ = shutdown_signal => {
                println!("Shutdown signal received, stopping acceptance of new connections");
                break;
            }
        }
    }
    
    println!("Server shutting down");
    Ok(())
}

async fn handle_connection(mut socket: tokio::net::TcpStream) -> Result<(), Box<dyn std::error::Error>> {
    let mut buffer = [0; 1024];
    
    loop {
        let n = socket.read(&mut buffer).await?;
        if n == 0 {
            break; // Connection closed
        }
        
        socket.write_all(&buffer[..n]).await?;
    }
    
    Ok(())
}
```

This pattern uses `tokio::select!` to wait for either a new connection or a shutdown signal. When a shutdown signal is received, we break out of the accept loop, allowing existing connections to complete before the application exits.

## Connection to rs-wsProxy

The rs-wsProxy project (a WebSocket to TCP proxy) demonstrates many of these concepts in a real-world application:

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::runtime::Builder;
use tokio::select;
use tokio::signal;
use tokio::sync::mpsc;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() {
    // Configure Tokio runtime with custom thread count
    let runtime = Builder::new_multi_thread()
        .worker_threads(4) // Use 4 worker threads
        .enable_all()
        .build()
        .unwrap();
    
    runtime.block_on(async {
        let addr = "0.0.0.0:8080";
        let listener = TcpListener::bind(addr).await.unwrap();
        println!("WebSocket proxy listening on {}", addr);
        
        // Set up shutdown signal handling
        let mut shutdown = signal::ctrl_c();
        
        loop {
            select! {
                // Accept new WebSocket connections
                res = listener.accept() => {
                    match res {
                        Ok((ws_stream, addr)) => {
                            println!("New WebSocket connection from: {}", addr);
                            // Spawn a task to handle each WebSocket connection
                            tokio::spawn(handle_websocket(ws_stream));
                        }
                        Err(e) => {
                            eprintln!("Failed to accept connection: {}", e);
                        }
                    }
                }
                // Handle shutdown signal
                _ = &mut shutdown => {
                    println!("Shutdown signal received");
                    break;
                }
            }
        }
        
        println!("Proxy shutting down gracefully");
    });
}

async fn handle_websocket(mut ws_stream: tokio_tungstenite::WebSocketStream<tokio::net::TcpStream>) {
    // WebSocket handshake is already complete when we get here
    
    // Connect to the target TCP server
    let target_addr = "127.0.0.1:9000"; // Example target
    match TcpStream::connect(target_addr).await {
        Ok(mut tcp_stream) => {
            println!("Connected to target TCP server at {}", target_addr);
            
            // Create channels for bidirectional communication
            let (ws_to_tx, mut ws_to_rx) = tokio::sync::mpsc::channel(32);
            let (tcp_to_tx, mut tcp_to_rx) = tokio::sync::mpsc::channel(32);
            
            // Spawn tasks for each direction of communication
            let ws_to_tcp = tokio::spawn(async move {
                while let Some(msg) = ws_to_rx.recv().await {
                    if let Err(e) = tcp_stream.write_all(&msg).await {
                        eprintln!("Failed to write to TCP stream: {}", e);
                        break;
                    }
                }
            });
            
            let tcp_to_ws = tokio::spawn(async move {
                let mut buffer = vec![0; 1024];
                while let Ok(n) = tcp_stream.read(&mut buffer).await {
                    if n == 0 {
                        break; // Connection closed
                    }
                    let data = buffer[..n].to_vec();
                    if let Err(e) = tcp_to_tx.send(data).await {
                        eprintln!("Failed to send to WebSocket: {}", e);
                        break;
                    }
                }
            });
            
            // WebSocket message handling task
            let ws_handler = tokio::spawn(async move {
                while let Some(Ok(msg)) = ws_stream.next().await {
                    if let Ok(data) = msg.into_data() {
                        if let Err(e) = ws_to_tx.send(data).await {
                            eprintln!("Failed to send to WS->TCP channel: {}", e);
                            break;
                        }
                    }
                }
            });
            
            // TCP to WebSocket forwarding task
            let tcp_forwarder = tokio::spawn(async move {
                while let Some(data) = tcp_to_rx.recv().await {
                    if let Err(e) = ws_stream.send(tokio_tungstenite::tungstenite::Message::Binary(data)).await {
                        eprintln!("Failed to send to WebSocket: {}", e);
                        break;
                    }
                }
            });
            
            // Wait for any task to complete (indicating connection end)
            select! {
                _ = ws_to_tcp => {}
                _ = tcp_to_ws => {}
                _ = ws_handler => {}
                _ = tcp_forwarder => {}
            }
            
            // Clean shutdown of TCP connection
            let _ = tcp_stream.shutdown().await;
        }
        Err(e) => {
            eprintln!("Failed to connect to target TCP server: {}", e);
            let _ = ws_stream.close().await;
        }
    }
}
```

This example demonstrates:
- Custom Tokio runtime configuration with 4 worker threads
- Using `tokio::select!` to handle multiple concurrent operations (accepting connections, shutdown signal)
- Spawning multiple tasks per WebSocket connection for bidirectional communication
- Using mpsc channels for communication between tasks
- Proper resource cleanup on connection termination

## Summary

Asynchronous programming with Tokio enables building high-performance network applications in Rust without the overhead of thread-per-connection models. We've covered:

1. **Why Async**: Efficiency for I/O-bound applications like network proxies
2. **async/await**: The foundation of async Rust programming
3. **Tokio Runtime**: Configurable runtime options for different workloads
4. **Spawning Tasks**: Running concurrent asynchronous operations
5. **Channels**: Communication between tasks (mpsc and oneshot)
6. **Async I/O**: Non-blocking network operations with TcpListener/TcpStream
7. **tokio::select!**: Handling multiple asynchronous operations concurrently
8. **Graceful Shutdown**: Properly handling shutdown signals for clean termination
9. **Real-world Application**: How these concepts apply to a WebSocket proxy (rs-wsProxy)

These concepts form the foundation for building efficient, scalable network services in Rust. The ws-proxy project demonstrates how to combine these techniques to create a production-ready WebSocket to TCP proxy that can handle thousands of concurrent connections with minimal resource overhead.

← Previous: [Traits & Generics](/posts/rust/rust-traits-generics/)
Next: [Building wsProxy — CLI, Config & Server](/posts/rust/rust-wsproxy-server/)