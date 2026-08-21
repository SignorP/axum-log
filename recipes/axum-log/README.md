# 📦 Axum Log Recipe

A zero-dependency, pluggable logging middleware recipe for your modular Axum backend. This recipe sets up a dedicated `common/middlewares` directory layer, allowing developers to inspect incoming HTTP requests, status codes, and execution latencies instantly without adding heavy third-party tracing binaries.

---

## 📂 Architecture & File Tree

This recipe builds smoothly on top of your core workspace, introducing a nested `common` layout designed for system-wide, shared utility behaviors.

```text
src/
├── main.rs
├── lib.rs
├── routes.rs          # Central gateway where the logging layer is mounted
├── common/            # Shared cross-cutting concerns layer
│   ├── mod.rs         # Exports internal subfolders (e.g., middlewares)
│   └── middlewares/
│       ├── mod.rs     # Registration hub for all custom middlewares
│       └── log_middleware.rs # Core logging function implementation
└── modules/
    └── health/        # Standalone feature modules
```

---

## 🛠️ Code Implementation

### 1. The Core Middleware (`src/common/middlewares/log_middleware.rs`)

Intercepts the request, benchmarks lifecycle metrics, logs details to the stdout terminal, and passes the context to the next internal node.

```rust
use axum::extract::Request;
use axum::middleware::Next;
use axum::response::Response;
use std::time::Instant;

pub async fn middleware(request: Request, next: Next) -> Response {
    let start = Instant::now();
    let method = request.method().clone();
    let uri = request.uri().clone();

    let response = next.run(request).await;

    let duration = start.elapsed();
    let status = response.status();

    println!("🎯 [{}] {} -> {} ({:?})", method, uri, status, duration);
    response
}
```

### 2. Global Integration (`src/routes.rs`)

Inject the logger globally across all structural endpoints via `axum::middleware::from_fn`. Because Axum execution flows from bottom to top, registering your layer at the end ensures all traffic hits the logging pipe first.

```rust
#[path = "common/mod.rs"]
mod common;
#[path = "modules/mod.rs"]
mod modules;
use axum::Router;

pub fn create_router() -> Router {
    Router::new()
        .nest("/api/health", modules::health::router())
        .layer(axum::middleware::from_fn(
            common::middlewares::log_middleware::middleware,
        ))
}
```

---

## 🚀 Execution & Sample Terminal Logs

Boot the engine without needing to pass external `RUST_LOG` parameters:

```bash
xnex install axum-base axum-log -y -s
cargo run
```

### Output Trace

Once your requests navigate through the middleware stack, the console prints isolated metrics tracking exact execution milestones:

```text
🚀 Server started: http://127.0.0.1:3001
🎯 [GET] / -> 404 Not Found (34.3µs)
🎯 [GET] /favicon.ico -> 404 Not Found (154µs)
🎯 [GET] /api/health -> 200 OK (130.3µs)

```
