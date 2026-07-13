---
title: "Building wsProxy — CLI, Config & Server"
subtitle: ""
date: 2026-07-13T09:00:00+07:00
lastmod: 2026-07-13T09:00:00+07:00
draft: false
author: "Kawin Viriyaprasopsook"
authorLink: "https://kawin.dev"
description: "Build a WebSocket-to-TCP proxy with clap, axum, and tokio — CLI args, config, logging, and the HTTP/WS server"
license: ""
images: []
tags: ["Rust", "Tutorial", "WebSocket", "Axum", "Tokio"]
categories: ["Rust"]
featuredImage: "featured-image.jpeg"
featuredImagePreview: "featured-image.jpeg"
lightgallery: true
---

<!--more-->

Time to build the real project! After seeing that Facebook post from [rayrag.com](https://rayrag.com/) showing RO playable in a browser via WebSocket, I wanted to build my own proxy. Now, after 6 parts of learning Rust fundamentals, we finally put it all together. wsProxy is a WebSocket-to-TCP proxy for [roBrowser](https://github.com/vthibault/roBrowser) — it bridges browser WebSocket clients to plain TCP game servers. Source code is available at [https://github.com/bouroo/rs-wsProxy](https://github.com/bouroo/rs-wsProxy).

## Project Structure

Let's first look at the project structure we'll build:

```
rs-wsProxy/
├── Cargo.toml
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── config.rs
│   ├── logging.rs
│   ├── modules.rs
│   ├── proxy.rs
│   └── server.rs
└── tests/
    └── integration_test.rs
```

Each module has a clear responsibility:
- `main.rs`: Application entry point, runtime setup, and graceful shutdown
- `lib.rs`: Library re-exports and shared types
- `config.rs`: CLI argument parsing with Clap
- `logging.rs`: Tracing subscriber setup
- `modules.rs`: Request validation logic (redirects and allow-lists)
- `proxy.rs`: Core WebSocket-to-TCP proxy logic (covered in Part 8)
- `server.rs`: Axum HTTP server and WebSocket handlers
- `tests/`: Integration tests

## Cargo.toml — Dependencies

Here's our `Cargo.toml` with explanations for each dependency:

```toml
[package]
name = "rs-wsproxy"
version = "0.1.0"
edition = "2021"
edition = "2021"

[dependencies]
# Axum web framework built on Tokio and Tower
axum = "0.7"
# TLS support for Axum (HTTPS/WSS)
axum-server = "0.4"
# Async runtime
tokio = { version = "1", features = ["full"] }
# Rustls TLS implementation
rustls = "0.21"
# Command-line argument parser
clap = { version = "4.4", features = ["derive"] }
# Structured logging
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "fmt"] }
# Byte manipulation for proxying data
bytes = "1.5"
# Async utilities
futures-util = "0.3"
# Development dependencies
[dev-dependencies]
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
```

**Why these dependencies?**
- **Axum**: Provides ergonomic routing and extraction built on Tower and Tokio
- **Axum-server**: Enables HTTPS/WSS support via rustls
- **Tokio**: The async runtime that powers everything
- **Rustls**: Modern TLS implementation (safer than OpenSSL)
- **Clap**: Derive-based CLI argument parser with excellent ergonomics
- **Tracing**: Structured, contextual logging with EnvFilter support
- **Bytes**: Efficient byte buffer handling for proxying data
- **Futures-util**: Essential async utilities like `StreamExt` and `SinkExt`

## CLI with clap (config.rs)

We define our CLI arguments in `src/config.rs` using Clap's derive macros:

```rust
use clap::Parser;
use std::net::SocketAddr;
use std::net::IpAddr;
use std::net::Ipv4Addr;
use std::net::Ipv6Addr;
use std::net::SocketAddrV4;
use std::net::SocketAddrV6;
use std::net::ToSocketAddrs;
use std::net::ToSocketAddrs;
use std::net::ToSocketAddrs;
use std::net::ToSocketAddrs;

/// WebSocket to TCP proxy server
#[derive(Parser, Debug)]
#[command(author, version, about, long_about = None)]
pub struct Args {
    /// Listen address (default: 0.0.0.0:8080)
    #[arg(long, default_value_t = SocketAddr::from(([0, 0, 0, 0], 8080)))]
    pub addr: SocketAddr,

    /// Number of Tokio worker threads (default: num_cpus)
    #[arg(long, default_value_t = num_cpus::get())]
    pub threads: usize,

    /// Enable SSL/TLS (wss:// and https://)
    #[arg(long)]
    pub ssl: bool,

    /// Path to SSL certificate file (required if ssl=true)
    #[arg(long)]
    pub cert: Option<String>,

    /// Path to SSL private key file (required if ssl=true)
    #[arg(long)]
    pub key: Option<String>,

    /// Comma-separated list of allowed target hosts (empty = deny all, unset = allow all)
    #[arg(long, value_delimiter = ',')]
    pub allow: Option<Vec<String>>,

    /// Comma-separated list of redirect rules (from=to,from2=to2)
    #[arg(long, value_delimiter = ','), value_parser = parse_redirect_pair)]
    pub redirect: Option<Vec<(String, String)>>,

    /// Default target host:port when none specified in WebSocket path
    #[arg(long)]
    pub default_target: Option<String>,
}

/// Parse a single "from=to" redirect pair
fn parse_redirect_pair(src: &str) -> Result<(String, String), String> {
    let mut parts = src.splitn(2, '=');
    let from = parts
        .next()
        .ok_or_else(|| "missing '=' in redirect pair")?
        .trim()
        .to_string();
    let to = parts
        .next()
        .ok_or_else(|| "missing '=' in redirect pair")?
        .trim()
        .to_string();
    Ok((from, to))
}
```

**Key Clap features explained:**
- `#[arg(long, default_value_t = ...)]`: Sets default values using Rust expressions
- `value_delimiter = ','`: Splits comma-separated values into a vector
- `value_parser = parse_redirect_pair`: Custom validator for redirect format
- `Option<Vec<String>>` for `allow`: `None` = allow all, `Some(vec![])` = deny all, `Some(vec![...])` = allow list
- Custom parser for redirect pairs ensures `from=to` format

## AppState — Shared State

Our application state is shared across all routes via Axum's extractor pattern:

```rust
use std::collections::HashMap;
use std::sync::Arc;

/// Application state shared across all request handlers
#[derive(Clone)]
pub struct AppState {
    /// None = allow all targets, Some(vec[]) = deny all, Some(vec[hosts]) = allow list
    pub allowed_servers: Option<Vec<String>>,
    /// Redirect mappings: incoming host -> target host
    pub redirects: HashMap<String, String>,
    /// Default target when WebSocket path doesn't specify host:port
    pub default_target: Option<String>,
}

impl AppState {
    /// Create new state from CLI arguments
    pub fn from_args(args: &crate::config::Args) -> Self {
        Self {
            allowed_servers: args.allow.as_ref().map(|v| v.clone()),
            redirects: args
                .redirect
                .as_ref()
                .map(|v| v.iter().cloned().collect())
                .unwrap_or_default(),
            default_target: args.default_target.clone(),
        }
    }
}
```

**Why `Option<Vec<String>>` for `allowed_servers`?**
- `None` (flag not provided): Open proxy - allow any target
- `Some(vec![])` (empty explicit list): Deny all targets
- `Some(vec![allowed hosts])`: Allow only specified hosts

This three-state design gives us fine-grained control:
1. Open proxy (default, convenient for dev)
2. Strict deny-all (secure by default)
3. Explicit allow-list (production use)

## Parsing Config

Helper functions in `config.rs` convert CLI arguments into usable data structures:

```rust
/// Build allowed server list from CLI arg
pub fn build_allowed_list(allow: &Option<Vec<String>>) -> Option<Vec<String>> {
    allow.as_ref().map(|v| {
        v.iter()
            .map(|s| s.trim().to_lowercase())
            .filter(|s| !s.is_empty())
            .collect()
    })
}

/// Build redirect map from CLI arg
pub fn build_redirects(redirect: &Option<Vec<(String, String)>>) -> std::collections::HashMap<String, String> {
    redirect
        .as_ref()
        .map(|v| {
            v.iter()
                .map(|(k, v)| {
                    (
                        k.trim().to_lowercase(),
                        v.trim().to_string(),
                    )
                })
                .collect()
        })
        .unwrap_or_default()
}
```

**Iterator chain explanation:**
1. `as_ref()`: Convert `&Option<T>` to `Option<&T>` to avoid moving
2. `.map(|v| ...)`: Transform the inner vector if present
3. `.iter()`: Iterate over each string
4. `.map(|s| ...)`: Transform each string (trim + lowercase)
5. `.filter(...)`: Remove empty strings
6. `.collect()`: Build new Vector

For redirects, we do similar processing but collect into a `HashMap` for O(1) lookups.

## Logging (logging.rs)

We configure tracing with environment variable support:

```rust
use tracing_subscriber::{fmt, EnvFilter, layer::SubscriberExt};

/// Initialize tracing subscriber with environment filter
pub fn init_logging() {
    // Format logs with target, level, and message
    fmt()
        .with_target(true)
        .with_level(true)
        .with_thread_ids(true)
        .with_target(true)
        // Enable filtering via RUST_LOG env var
        .with_env_filter(EnvFilter::from_default_env())
        .init();
}
```

**Usage:** 
- Set `RUST_LOG=info` for general info logs
- Set `RUST_LOG=wsproxy=debug,axum::rejection=trace` for detailed debugging
- Automatic JSON output in JSON-logging environments (like Kubernetes)
- Thread IDs help debug concurrency issues

## Axum Server (server.rs)

Our Axum server setup with WebSocket upgrade handlers:

```rust
use axum::{
    extract::{State, WebSocketUpgrade},
    response::Response,
    routing::get,
    Router,
};
use std::net::SocketAddr;
use std::sync::Arc;
use tokio::net::TcpListener;
use tracing::{info, info_span};

use crate::{AppState, modules::verify, proxy::handle_proxy};

/// Create Axum router with shared state
pub fn app(state: Arc<AppState>) -> Router {
    Router::new()
        .route("/", get(|| async { "WebSocket to TCP Proxy" }))
        .route("/ws/:target", get(ws_handler))
        .route("/:target", get(ws_handler))
        .with_state(state)
}

/// WebSocket upgrade handler
async fn ws_handler(
    State(state): State<Arc<AppState>>,
    ws: WebSocketUpgrade,
    // Extract target from path: /ws/example.com:8080 or /example.com:8080
    axum::extract::Path(target): axum::extract::Path<String>,
) -> Response {
    // Validate and normalize target
    let target = match verify(&state, &target) {
        Ok(t) => t,
        Err(e) => {
            return Response::builder()
                .status(400)
                .body(e.into())
                .unwrap();
        }
    };

    ws.on_upgrade(move |socket| handle_proxy(socket, target))
}

/// Start the server
pub async fn run(addr: SocketAddr, state: Arc<AppState>) {
    let listener = TcpListener::bind(addr).await.expect("Failed to bind");
    info!(%addr, "server listening");

    axum::serve(listener, app(state))
        .await
        .expect("server error");
}
```

**Key points:**
- Two routes: `/` (health check) and `/:target` or `/ws/:target` (WebSocket)
- `State<Arc<AppState>>` shares application state efficiently
- `WebSocketUpgrade` extractor handles WebSocket handshake
- Path extraction gives us the target host:port
- Validation happens before WebSocket upgrade
- `on_upgrade` converts established WebSocket to our proxy handler

## Verify Pipeline (modules.rs)

Request validation happens in two phases:

```rust
use std::collections::HashMap;

/// Validate and normalize target host:port
pub fn validate_target(target: &str) -> Result<String, String> {
    // Must contain colon for host:port
    if !target.contains(':') {
        return Err(format!("target must be in host:port format, got '{target}'"));
    }

    let parts: Vec<&str> = target.splitn(2, ':').collect();
    let host = parts[0].to_lowercase();
    let port = parts[1];

    // Validate port is numeric and in valid range
    let port: u16 = port
        .parse()
        .map_err(|_| format!("port must be a number, got '{port}'"))?;
    if port == 0 || port > 65535 {
        return Err(format!("port must be between 1-65535, got {port}"));
    }

    Ok(format!("{host}:{port}"))
}

/// Apply redirect rules then check allow-list
pub fn verify(state: &super::AppState, target: &str) -> Result<String, String> {
    // Step 1: Apply redirect rules (if any)
    let redirected_target = state
        .redirects
        .get(target)
        .map(|t| t.as_str())
        .unwrap_or(target);

    // Step 2: Validate format (host:port)
    let validated_target = validate_target(redirected_target)?;

    // Step 3: Extract host portion for allow-list check
    let host = validated_target
        .split(':')
        .next()
        .ok_or_else(|| "invalid target format".to_string())?
        .to_lowercase();

    // Step 4: Check allow-list (None = allow all, Some([]) = deny all)
    match &state.allowed_servers {
        None => Ok(validated_target), // Allow all
        Some(list) if list.is_empty() => Err(format!("target '{host}' not allowed (empty allow-list)")),
        Some(list) => {
            if list.contains(&host) {
                Ok(validated_target)
            } else {
                Err(format!("target '{host}' not in allow-list"))
            }
        }
    }
}
```

**Pipeline explanation:**
1. **Redirect check**: If target matches a redirect rule, use the destination
2. **Format validation**: Ensure `host:port` format with valid port number
3. **Host extraction**: Split to get hostname for allow-list checking
4. **Allow-list check**: 
   - `None` (flag not set): Allow any target
   - `Some([])` (empty explicit list): Deny all targets
   - `Some([hosts])`: Only allow listed hosts

## main.rs — Entry Point

Application entry point with graceful shutdown:

```rust
use std::net::SocketAddr;
use std::sync::Arc;
use std::time::Duration;
use tokio::signal;
use tracing::info;

mod config;
mod logging;
mod modules;
mod proxy;
mod server;

use crate::{config::Args, server::AppState};

#[tokio::main]
async fn main() {
    // Initialize logging first
    logging::init_logging();

    // Parse command line arguments
    let args = Args::parse();
    info!(?args, "starting wsproxy");

    // Build application state
    let state = Arc::new(AppState::from_args(&args));

    // Configure server address
    let addr = args.addr;

    // Start server
    let server_task = tokio::spawn(async move {
        server::run(addr, state).await;
    });

    // Wait for shutdown signal
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => info!("received Ctrl+C, shutting down"),
        _ = terminate => info!("received terminate signal, shutting down"),
    }

    // Graceful shutdown: wait for server task to finish
    let _ = server_task.await;
    info!("server stopped");
}
```

**Key features:**
- `#[tokio::main]` macro sets up multi-threaded runtime
- Logging initialized before anything else
- Arguments parsed into strongly-typed `Args` struct
- Application state wrapped in `Arc` for cheap cloning
- Server runs in separate task so we can wait for shutdown signals
- Handles both Ctrl+C (SIGINT) and SIGTERM (Unix)
- Graceful shutdown waits for server task to complete

## Testing

Run and test the proxy:

```bash
# Run with default settings (HTTP on port 8080, open proxy)
cargo run --release

# Test with HTTP client (returns plain text)
curl http://localhost:8080/
# Output: WebSocket to TCP Proxy

# Test WebSocket endpoint (requires WebSocket client)
# Using websocat:
# echo "hello" | websocat ws://localhost:8080/echo.websocket.org:80

# With SSL (requires cert/key files):
cargo run -- --ssl --cert ./cert.pem --key ./key.pem

# With allow-list (only allow specific hosts):
cargo run -- --allow=echo.websocket.org,example.com:80

# With redirects (redirect traffic from legacy.example.com to new.example.com):
cargo run -- --redirect=legacy.example.com:80=new.example.com:80

# With default target (when no target specified in path):
cargo run -- --default-target=echo.websocket.org:80
```

**Testing with websocat:**
```bash
# Send message through proxy to echo.websocket.org:80
echo "Hello WebSocket" | websocat ws://localhost:8080/echo.websocket.org:80

# Should receive echo response from server
```

**Expected behavior:**
- Root endpoint (`/`) returns plain text description
- WebSocket endpoints (`/ws/<host:port>` or `/<host:port>`) establish proxy connection
- Invalid targets return HTTP 400 with error message
- Redirects rewrite target before validation
- Allow-list blocks non-matching hosts (when enabled)

## Summary

We've built a functional WebSocket-to-TCP proxy with:
- Production-ready CLI using Clap with smart defaults
- Flexible configuration via allow-lists, redirects, and default targets
- Structured logging with Tracy and EnvFilter support
- Axum-based HTTP/WebSocket server with graceful shutdown
- Two-stage validation pipeline (redirects → validation → allow-listing)
- Proper state sharing via Arc<AppState> pattern

**Next steps:** In Part 8 we'll implement the core proxy logic in `proxy.rs` that actually bridges WebSocket and TCP connections, handle binary/data transfer, and add connection pooling.

**Links:**
← Previous: [Async Rust with Tokio](/posts/rust/rust-async-tokio/)
Next: [Building wsProxy — Proxy Core & Deployment](/posts/rust/rust-wsproxy-proxy-deploy/)
Source: https://github.com/bouroo/rs-wsProxy