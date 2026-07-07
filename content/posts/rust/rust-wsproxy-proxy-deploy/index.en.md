---
title: "Building wsProxy — Proxy Core & Deployment"
subtitle: ""
date: 2026-07-14T09:00:00+07:00
lastmod: 2026-07-14T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "The final pieces of wsProxy: TCP connection, bidirectional pump, TLS, testing, Docker, and Kubernetes deployment"
license: ""
images: []
tags: ["Rust", "Tutorial", "WebSocket", "Tokio", "Docker"]
categories: ["Rust"]
featuredImage: "/img/featured-image.webp"
featuredImagePreview: "/img/featured-image.webp"
lightgallery: true
---

# Building wsProxy — Proxy Core & Deployment

The final chapter! From that Facebook post by [rayrag.com](https://rayrag.com/) showing RO playable in a browser, to revisiting Rust over 7 parts, to building the CLI, config, and server — we now complete the proxy's heart: the TCP connection logic and bidirectional data pump that bridges WebSocket and TCP streams. We'll also cover TLS encryption, testing, and production deployment with Docker and Kubernetes.

<!--more-->

In [Part 7](/posts/rust/rust-wsproxy-server/), we built the HTTP server, WebSocket upgrade handler, and configuration system. Source: [https://github.com/bouroo/rs-wsProxy](https://github.com/bouroo/rs-wsProxy)

## TCP Connection (proxy.rs)

The proxy's heart is the TCP connection establishment and data pumping mechanism. Let's start with the `connect_tcp` function that resolves hostnames and establishes TCP connections:

```rust
use tokio::net::{TcpStream, lookup_host};
use std::net::IpAddr;

/// Establishes a TCP connection to the specified address.
///
/// Performs DNS lookup, filters for IPv4 addresses, and sets TCP_NODELAY.
pub async fn connect_tcp(addr: &str) -> Result<TcpStream, String> {
    // Resolve the address to IP endpoints
    let addrs: Vec<_> = lookup_host(addr)
        .await
        .map_err(|e| format!("DNS lookup failed for '{}': {}", addr, e))?
        .filter(|a| {
            // Only keep IPv4 addresses for compatibility
            matches!(a, IpAddr::V4(_))
        })
        .collect();

    // Ensure we found at least one IPv4 address
    if addrs.is_empty() {
        return Err(format!("no IPv4 address found for '{}'", addr));
    }

    // Try each address until we establish a connection
    let mut last_err = None;
    for addr in addrs {
        match TcpStream::connect(&addr).await {
            Ok(stream) => {
                // Disable Nagle's algorithm for low-latency proxying
                if let Err(e) = stream.set_nodelay(true) {
                    return Err(format!("failed to set TCP_NODELAY: {}", e));
                }
                return Ok(stream);
            }
            Err(e) => last_err = Some(e),
        }
    }

    Err(format!(
        "failed to connect to {} after trying all resolved addresses: {}",
        addr,
        last_err.map(|e| e.to_string()).unwrap_or_else(|| "unknown error".to_string())
    ))
}
```

### Key Details Explained

**DNS Resolution and IPv4 Filtering**: We use `tokio::net::lookup_host` to perform asynchronous DNS resolution. This returns all IP addresses (both IPv4 and IPv6) associated with the hostname. Since our target environments primarily use IPv4 and to avoid potential compatibility issues, we filter for IPv4 addresses only using `matches!(a, IpAddr::V4(_))`. If no IPv4 addresses are found, we return a clear error.

**Connection Attempt Loop**: We iterate through each resolved IP address, attempting to establish a TCP connection. This provides resilience against partial DNS failures - if one IP address is unreachable, we try the next. The loop continues until either a connection succeeds or we exhaust all addresses.

**TCP_NODELAY Optimization**: After establishing a connection, we set the TCP_NODELAY socket option to disable Nagle's algorithm. This is crucial for proxy applications where low latency is important - we want small packets to be sent immediately rather than being buffered for potential aggregation. Without this setting, interactive protocols (like the roBrowser protocol wsProxy serves) would experience noticeable delays.

**Error Propagation**: Throughout the function, we convert low-level IO errors into descriptive String errors that include context about what operation failed and the target address. This makes debugging connection issues much simpler in production environments.

## Bidirectional Pump — The Concept

With a TCP connection established, we need to move data bidirectionally between the WebSocket and TCP streams. Unlike unidirectional data flow, a proxy must handle simultaneous data movement in both directions:

1. **WebSocket → TCP**: Messages received from the WebSocket client must be forwarded to the TCP backend
2. **TCP → WebSocket**: Data received from the TCP backend must be sent to the WebSocket client

These operations are independent and can occur concurrently - the WebSocket might receive a message while we're still sending previous TCP data, and vice versa. Using sequential blocking operations would create head-of-line blocking where one direction stalls the other.

Tokio's `select!` macro provides the perfect solution: it allows us to run multiple asynchronous operations concurrently and proceed as soon as any one completes. We'll create two asynchronous tasks - one for each direction - and use `select!` to wait for either to complete (indicating the connection should close).

## WS → TCP Direction

Let's examine the WebSocket-to-TCP data flow. The roBrowser protocol uses binary WebSocket frames exclusively, so we filter for `Message::Binary` variants:

```rust
use tokio_tungstenite::WebSocketStream;
use tokio_tungstenite::tungstenite::protocol::Message;
use tokio::io::{self, AsyncWriteExt};

/// Forwards data from WebSocket to TCP stream.
async fn ws_to_tcp(
    mut ws_rx: tokio_tungstenite::WebSocketStream<tokio::net::TcpStream>,
    mut tcp_write: impl AsyncWriteExt + Unpin,
) -> Result<(), String> {
    while let Some(msg) = ws_rx.next().await {
        match msg {
            Ok(Message::Binary(data)) => {
                // Forward binary WebSocket frames to TCP
                if tcp_write.write_all(&data).await.is_err() {
                    break;
                }
                // Important: flush to ensure data is sent immediately
                if tcp_write.flush().await.is_err() {
                    break;
                }
            }
            Ok(Message::Close(_)) => {
                // WebSocket initiated close - forward to TCP then break
                let _ = tcp_write.shutdown().await;
                break;
            }
            Err(e) => return Err(format!("WebSocket receive error: {}", e)),
            _ => {} // Ignore other message types (text, ping, pong)
        }
    }
    Ok(())
}
```

### WS → TCP Mechanics

**Binary Frame Filtering**: We explicitly handle only `Message::Binary` variants because the roBrowser protocol encodes all data as binary frames. Text frames, ping/pong, and other WebSocket control frames are ignored for data forwarding (though we handle Close frames specially).

**Immediate Flushing**: After writing data to the TCP stream, we call `flush()` to ensure bytes are sent immediately rather than buffered. This minimizes latency for interactive applications.

**Close Frame Handling**: When we receive a WebSocket Close frame, we initiate a TCP shutdown (sending FIN packet) before breaking the loop. This allows the TCP backend to gracefully detect connection termination.

**Error Propagation**: Any error in receiving WebSocket messages or writing to TCP immediately terminates the forwarding loop and returns an error.

## TCP → WS Direction

The TCP-to-WebSocket direction requires more sophisticated buffer management to efficiently handle streaming data:

```rust
use tokio::io::{self, AsyncReadExt};
use bytes::{BytesMut, Buf};
use tokio_tungstenite::WebSocketStream;
use tokio_tungstenite::tungstenite::protocol::Message;

/// Forwards data from TCP to WebSocket stream.
async fn tcp_to_ws(
    mut tcp_read: impl AsyncReadExt + Unpin,
    mut ws_tx: tokio::sync::mpsc::UnboundedSender<Message>,
) -> Result<(), String> {
    // Allocate buffer with reasonable initial capacity
    let mut buf = BytesMut::with_capacity(65536);

    loop {
        // Ensure buffer has sufficient capacity for read operations
        if buf.capacity() < 4096 {
            buf.reserve(65536 - buf.capacity());
        }

        // Read data from TCP into our buffer
        match tcp_read.read_buf(&mut buf).await {
            // Zero bytes indicates TCP stream closed
            Ok(0) => {
                // Send close frame to WebSocket client
                let _ = ws_tx.send(Message::Close(None)).await;
                break;
            }
            Ok(n) => {
                // Extract all available data as a contiguous byte slice
                let data = buf.split().freeze();
                
                // Attempt to send data to WebSocket
                if ws_tx.send(Message::Binary(data)).await.is_err() {
                    // Send failed - likely WebSocket closed
                    break;
                }
                // Buffer automatically reused for next read
            }
            Err(e) => return Err(format!("TCP read error: {}", e)),
        }
    }
    Ok(())
}
```

### TCP → WS Buffer Management

**BytesMut for Efficient Buffering**: We use `bytes::BytesMut` which provides mutable byte buffer with efficient resizing and zero-copy operations. Starting with 64KB capacity balances memory usage with reducing reallocation frequency.

**Dynamic Capacity Management**: Before each read operation, we ensure the buffer has at least 4KB free space. If not, we reserve additional capacity, we reserve another 64KB. This prevents frequent small reallocations while bounding memory growth.

**Zero-Copy Data Extraction**: The `split()` method extracts all buffered data as a new `BytesMut` instance without copying, and `freeze()` converts it to an immutable `Bytes` that's cheap to clone. This avoids data duplication when sending to the WebSocket sender.

**Read Loop Semantics**: 
- `read_buf` returns `Ok(0)` when the TCP stream is closed (EOF)
- For positive byte counts, we extract and forward all buffered data
- Any I/O error breaks the loop and propagates upward

This design ensures efficient memory usage while maintaining low-latency data forwarding - critical for proxy performance.

## Stream Splitting

When we establish a TCP connection, we immediately split it into separate read and write halves:

```rust
let (tcp_read, tcp_write) = tcp_stream.into_split();
```

### Why Split Streams?

Tokio's `TcpStream` internally uses a mutex-like BiLock to synchronize access between read and write operations. When we perform concurrent read and write operations on the same stream, they contend for this lock, creating unnecessary synchronization overhead.

By splitting the stream into `owned ReadHalf` and `owned WriteHalf`, we eliminate this contention:
- Each half can be used independently without locking
- Multiple tasks can read and write concurrently with zero synchronization cost
- Each half owns its portion of the resource and will close the connection when dropped

This optimization is particularly important in our proxy where we have two concurrent data pumps (WS→TCP and TCP→WS) operating simultaneously on the same TCP connection.

## TLS with rustls

For secure connections, wsProxy supports TLS termination using the rustls library. Here's how we configure the TLS acceptor:

```rust
use rustls::{ServerConfig, RootCertStore};
use rustls_pemfile::{certs, pkcs8_private_keys};
use std::fs::File;
use std::io::BufReader;
use axum::extract::State;
use axum::response::Response;
use axum::http::StatusCode;

/// Loads TLS configuration from PEM-encoded certificate and key files.
pub fn load_tls_config(cert_path: &str, key_path: &str) -> Result<ServerConfig, String> {
    // Load certificate chain
    let cert_file = File::open(cert_path)
        .map_err(|e| format!("Failed to open certificate file '{}': {}", cert_path, e))?;
    let cert_reader = BufReader::new(cert_file);
    let cert_chain = certs(cert_reader)
        .map_err(|e| format!("Failed to parse certificates: {}", e))?;
    
    // Load private key
    let key_file = File::open(key_path)
        .map_err(|e| format!("Failed to open key file '{}': {}", key_path, e))?;
    let key_reader = BufReader::new(key_file);
    let keys = pkcs8_private_keys(key_reader)
        .map_err(|e| format!("Failed to parse private key: {}", e))?;
    
    if keys.is_empty() {
        return Err("No private keys found in key file".to_string());
    }
    
    // Configure TLS server
    let mut config = ServerConfig::builder_with_root_certificates(RootCertStore::empty())
        .with_no_client_auth()
        .with_single_cert(cert_chain, keys[0].clone())
        .map_err(|e| format!("Failed to build TLS config: {}", e))?;
    
    // Optional: configure ALPN for HTTP/2 and HTTP/1.1
    config.alpn_protocols = vec![b"h2".to_vec(), b"http/1.1".to_vec()];
    
    Ok(config)
}
```

### TLS Integration Points

**Certificate Loading**: We use `rustls_pemfile` to parse PEM-encoded certificates and private keys. This approach avoids blocking the async runtime during TLS setup by performing file I/O during initialization.

**rustls Configuration**: We build a `ServerConfig` with no client authentication (typical for server-side TLS) and provide our certificate chain and private key. The ALPN protocols enable efficient protocol negotiation for HTTP/2 fallback.

**axum-server Integration**: In our HTTP server setup, we conditionally create a TLS acceptor when TLS configuration is provided:

```rust
let listener = if let Some(tls_config) = &tls_config {
    let tls_acceptor = tokio_rustls::TcpAcceptor::from(tls_config.clone());
    tokio::net::TcpListener::bind(&tls_bind_addr)
        .await
        .map_err(|e| format!("Failed to bind TLS listener: {}", e))?
        .into_tls(tls_acceptor)
} else {
    tokio::net::TcpListener::bind(&http_bind_addr)
        .await
        .map_err(|e| format!("Failed to bind HTTP listener: {}", e))?
};
```

This allows the same handler code to work for both plain HTTP and HTTPS connections, with axum-server handling the TLS transparently.

## Graceful Shutdown

Production services must handle shutdown signals gracefully to prevent data loss and allow active connections to complete. Here's our signal handling implementation:

```rust
use tokio::signal;
use tokio::time::{timeout, Duration};
use futures::future::join_all;

/// Handles graceful shutdown on SIGINT and SIGTERM.
async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };
    
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };
    
    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
    
    info!("Shutdown signal received, starting graceful shutdown...");
}

/// Performs graceful shutdown with timeout for active connections.
async fn graceful_shutdown(
    server: Server,
    shutdown_timeout: Duration,
) {
    // Stop accepting new connections
    server.close();
    
    // Wait for active connections to complete, with timeout
    match timeout(shutdown_timeout, server.wait_for_rest()).await {
        Ok(_) => info!("All connections closed gracefully"),
        Err(_) => {
            warn!("Timeout waiting for connections to close, forcing shutdown");
        }
    }
}
```

### Shutdown Workflow

**Signal Handling**: We listen for both SIGINT (Ctrl+C) and SIGTERM (termination signal from orchestrators like Kubernetes) using Tokio's signal module. The `select!` waits for either signal.

**Two-Phase Shutdown**:
1. **Stop accepting new connections**: Call `server.close()` on the underlying TCP listener
2. **Drain existing connections**: Wait for active connections to complete using `server.wait_for_rest()` with a configurable timeout

This approach ensures we don't abruptly terminate active proxy sessions while still enforcing a maximum shutdown duration to prevent hanging processes.

## Testing

Comprehensive testing validates both individual components and end-to-end behavior. Let's look at our test strategy:

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tokio::net::TcpListener;
    use std::net::SocketAddr;

    #[tokio::test]
    async fn test_connect_tcp_success() {
        // Start a simple TCP echo server for testing
        let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
        let addr = listener.local_addr().unwrap();
        tokio::spawn(async move {
            if let Ok((mut socket, _)) = listener.accept().await {
                let mut buf = [0; 1024];
                while let Ok(n) = socket.read(&mut buf).await {
                    if n == 0 { break; }
                    let _ = socket.write_all(&buf[..n]).await;
                }
            }
        });

        // Test connection to our test server
        let result = connect_tcp(&addr.to_string()).await;
        assert!(result.is_ok(), "Failed to connect to test server: {}", result.unwrap_err());
        
        // Clean up
        let _ = listener.shutdown().await;
    }

    #[tokio::test]
    async fn test_connect_tcp_ipv4_only() {
        // Test that we only get IPv4 addresses even if IPv6 is available
        let result = connect_tcp("localhost").await;
        // Should succeed with IPv4 localhost (127.0.0.1)
        assert!(result.is_ok());
    }
}
```

### Integration Tests

Our integration tests verify the complete WebSocket-to-TCP proxy pipeline:

```rust
#[tokio::test]
async fn test_ws_to_tcp_proxy() {
    // 1. Start TCP echo server
    let tcp_listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let tcp_addr = tcp_listener.local_addr().unwrap();
    let tcp_echo_task = tokio::spawn(async move {
        if let Ok((mut socket, _)) = tcp_listener.accept().await {
            let mut buf = [0; 1024];
            while let Ok(n) = socket.read(&mut buf).await {
                if n == 0 { break; }
                let _ = socket.write_all(&buf[..n]).await;
            }
        }
    });

    // 2. Start WebSocket server (our proxy frontend)
    // ... (setup code omitted for brevity)

    // 3. Connect WebSocket client and exchange data
    // ... (test WebSocket connection and data transfer)

    // 4. Cleanup
    tcp_echo_task.abort();
    let _ = tcp_listener.shutdown().await;
}
```

These tests verify:
- Individual components (`connect_tcp` function)
- End-to-end data flow (WebSocket client → proxy → TCP server → proxy → WebSocket client)
- Error handling and connection termination sequences
- TLS handshake when enabled

## Docker Deployment

Containerizing wsProxy enables consistent deployment across environments. Here's our multi-stage Dockerfile:

```dockerfile
# syntax = docker/dockerfile:1.4

# ------ Build Stage ------
FROM rust:1.75-slim-bullseye AS builder

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Create app directory
WORKDIR /app

# Cache dependencies by copying Cargo files first
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo "fn main() {}" > src/main.rs
RUN cargo build --release
RUN rm -rf src

# Copy actual source code
COPY . .

# Build application in release mode
RUN touch src/main.rs
RUN cargo build --release

# ------ Runtime Stage ------
FROM debian:bullseye-slim

# Install runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN useradd -m -u 1000 appuser
WORKDIR /app

# Copy built binary from builder stage
COPY --from=builder /app/target/release/wsproxy .

# Use non-root user
USER appuser

# Expose ports
EXPOSE 8080
EXPOSE 8443

# Entry point
ENTRYPOINT ["./wsproxy"]
```

### Docker Compose for Local Development

```yaml
version: '3.8'

services:
  wsproxy:
    build: .
    ports:
      - "8080:8080"
      - "8443:8443"
    environment:
      - WS_PROXY_TLS_CERT=/certs/fullchain.pem
      - WS_PROXY_TLS_KEY=/certs/privkey.pem
      - WS_PROXY_LOG_LEVEL=info
    volumes:
      - ./certs:/certs:ro
      - ./config.toml:/app/config.toml:ro
    restart: unless-stopped
```

The multi-stage build separates compilation dependencies from the runtime image, resulting in a minimal ~50MB final container. We run as a non-root user for security and expose both HTTP (8080) and HTTPS (8443) ports.

## Kubernetes Deployment

For production orchestration, we provide Kubernetes manifests:

### Deployment Manifest (`deploy/k8s/deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wsproxy
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
        image: bouroo/wsproxy:latest
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8443
          name: https
        env:
        - name: WS_PROXY_LOG_LEVEL
          value: "info"
        - name: WS_PROXY_TARGET_HOST
          value: "backend-service"
        - name: WS_PROXY_TARGET_PORT
          value: "80"
        volumeMounts:
        - name: tls-certs
          mountPath: /certs
          readOnly: true
        - name: config
          mountPath: /app/config.toml
          subPath: config.toml
          readOnly: true
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
      volumes:
      - name: tls-certs
        secret:
          secretName: wsproxy-tls
      - name: config
        configMap:
          name: wsproxy-config
```

### Service Manifest (`deploy/k8s/service.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wsproxy
  labels:
    app: wsproxy
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: https
    port: 443
    targetPort: 8443
  selector:
    app: wsproxy
```

This deployment includes:
- Horizontal pod autoscaling readiness (3 replicas)
- TLS termination via Kubernetes secrets
- Liveness and readiness probes for health checking
- Resource requests and limits for QoS
- ConfigMap for configuration separation
- LoadBalancer service for external access

## CI/CD Pipeline

Our GitHub Actions workflow automates testing, building, and deployment:

### CI Workflow (`.github/workflows/ci.yml`)

```yaml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Install Rust toolchain
      uses: dtolnay/rust-toolchain@stable
      with:
        components: rustfmt, clippy
    
    - name: Cache Cargo dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.cargo/registry
          ~/.cargo/git
          target
        key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
        restore-keys: |
          ${{ runner.os }}-cargo-
    
    - name: Run cargo fmt --check
      run: cargo fmt --check
    
    - name: Run cargo clippy
      run: cargo clippy -- -D warnings
    
    - name: Run tests
      run: cargo test -- --test-threads=1
    
    - name: Build release binary
      run: cargo build --release
```

### CD Workflow (`.github/workflows/cd.yml`)

```yaml
name: CD

on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Log in to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: bouroo/wsproxy:latest, bouroo/wsproxy:${{ github.sha }}
    
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'
    
    - name: Configure AWS EKS
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1
    
    - name: Update kubeconfig
      run: aws eks update-kubeconfig --name wsproxy-cluster
    
    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/wsproxy wsproxy=bouroo/wsproxy:${{ github.sha }}
        kubectl rollout status deployment/wsproxy
```

The CI pipeline ensures code quality through formatting checks, linting, and testing. The CD pipeline builds Docker images, pushes to a registry, and performs rolling updates to Kubernetes clusters.

## Series Conclusion

Over eight articles, we've journeyed from Rust fundamentals to a production-ready WebSocket proxy:

1. **[Part 1: Getting Started with Rust](/posts/rust/rust-getting-started/)** — Installation, Cargo, variables, data types, functions, control flow
2. **[Part 2: Ownership, Borrowing & Lifetimes](/posts/rust/rust-ownership-borrowing/)** — The ownership model, references, slices, lifetimes
3. **[Part 3: Structs, Enums & Pattern Matching](/posts/rust/rust-structs-enums/)** — Custom types, Option, match, Result intro
4. **[Part 4: Collections, Iterators & Error Handling](/posts/rust/rust-collections-errors/)** — Vec, HashMap, iterator adaptors, the `?` operator
5. **[Part 5: Traits & Generics](/posts/rust/rust-traits-generics/)** — Trait definitions, trait bounds, trait objects, standard traits
6. **[Part 6: Async Rust with Tokio](/posts/rust/rust-async-tokio/)** — async/await, runtime, tasks, channels, async I/O, `select!`
7. **[Part 7: Building wsProxy — CLI, Config & Server](/posts/rust/rust-wsproxy-server/)** — clap, axum router, WebSocket upgrade, verify pipeline
8. **Part 8: Proxy Core & Deployment (This Article)** — TCP connection, bidirectional pump, TLS, testing, Docker/K8s

We've covered the complete stack:
- **Networking**: TCP/UDP, DNS resolution, TLS encryption
- **Asynchronous Rust**: Tokio runtime, futures, streams, and buffers
- **WebSocket Protocol**: Frame handling, binary data transfer, connection lifecycle
- **HTTP Server**: Routing, middleware, and integration with WebSockets
- **Production Concerns**: Testing strategies, containerization, orchestration, and observability

The wsProxy project demonstrates how Rust's zero-cost abstractions, memory safety, and excellent async ecosystem enable building high-performance network services that rival C/C++ implementations while providing superior developer productivity and safety.

Looking back, it all started with a simple Facebook post from [rayrag.com](https://rayrag.com/) showing RO running in a browser. That single moment of curiosity led to revisiting Rust and building a production-ready proxy. That's the beauty of programming — curiosity leads to learning, and learning leads to creation.

The complete source code is available at [https://github.com/bouroo/rs-wsProxy](https://github.com/bouroo/rs-wsProxy). Thank you for following along this journey — happy coding!

[← Previous: Building wsProxy — CLI, Config & Server](/posts/rust/rust-wsproxy-server/)
Source: https://github.com/bouroo/rs-wsProxy