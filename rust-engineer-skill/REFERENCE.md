# Rust Engineer - Technical Reference

This document contains detailed workflows, technical specifications, and advanced patterns for Rust development.

## Workflow: Implement Actor Model with Tokio Channels

**Goal:** Build concurrent actor system for stateful processing with message passing.

### Step 1: Define Actor Messages

```rust
use tokio::sync::oneshot;
use uuid::Uuid;

pub enum OrderMessage {
    Create {
        request: CreateOrderRequest,
        response: oneshot::Sender<Result<OrderResponse, OrderError>>,
    },
    Get {
        id: Uuid,
        response: oneshot::Sender<Result<OrderResponse, OrderError>>,
    },
    UpdateStatus {
        id: Uuid,
        status: String,
        response: oneshot::Sender<Result<(), OrderError>>,
    },
}
```

### Step 2: Implement Actor

```rust
use tokio::sync::mpsc;
use std::collections::HashMap;

pub struct OrderActor {
    receiver: mpsc::UnboundedReceiver<OrderMessage>,
    db: PgPool,
    cache: HashMap<Uuid, OrderResponse>,
}

impl OrderActor {
    pub fn new(receiver: mpsc::UnboundedReceiver<OrderMessage>, db: PgPool) -> Self {
        Self {
            receiver,
            db,
            cache: HashMap::new(),
        }
    }
    
    pub async fn run(mut self) {
        while let Some(message) = self.receiver.recv().await {
            self.handle_message(message).await;
        }
    }
    
    async fn handle_message(&mut self, message: OrderMessage) {
        match message {
            OrderMessage::Create { request, response } => {
                let result = service::create_order(&self.db, request).await;
                if let Ok(ref order) = result {
                    self.cache.insert(order.id, order.clone());
                }
                let _ = response.send(result);
            }
            OrderMessage::Get { id, response } => {
                let result = if let Some(cached) = self.cache.get(&id) {
                    Ok(cached.clone())
                } else {
                    service::get_order(&self.db, id).await
                };
                let _ = response.send(result);
            }
            OrderMessage::UpdateStatus { id, status, response } => {
                let result = update_order_status(&self.db, id, &status).await;
                if result.is_ok() {
                    if let Some(order) = self.cache.get_mut(&id) {
                        order.status = status;
                    }
                }
                let _ = response.send(result);
            }
        }
    }
}
```

### Step 3: Create Client Handle

```rust
#[derive(Clone)]
pub struct OrderClient {
    sender: mpsc::UnboundedSender<OrderMessage>,
}

impl OrderClient {
    pub fn new(sender: mpsc::UnboundedSender<OrderMessage>) -> Self {
        Self { sender }
    }
    
    pub async fn create_order(
        &self,
        request: CreateOrderRequest,
    ) -> Result<OrderResponse, OrderError> {
        let (tx, rx) = oneshot::channel();
        self.sender
            .send(OrderMessage::Create { request, response: tx })
            .map_err(|_| OrderError::InvalidData("Actor dropped".to_string()))?;
        rx.await
            .map_err(|_| OrderError::InvalidData("Response dropped".to_string()))?
    }
    
    pub async fn get_order(&self, id: Uuid) -> Result<OrderResponse, OrderError> {
        let (tx, rx) = oneshot::channel();
        self.sender
            .send(OrderMessage::Get { id, response: tx })
            .map_err(|_| OrderError::InvalidData("Actor dropped".to_string()))?;
        rx.await
            .map_err(|_| OrderError::InvalidData("Response dropped".to_string()))?
    }
}
```

### Step 4: Spawn Actor in Main

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let db = PgPool::connect(&database_url).await?;
    
    let (tx, rx) = mpsc::unbounded_channel();
    let actor = OrderActor::new(rx, db.clone());
    
    // Spawn actor task
    tokio::spawn(async move {
        actor.run().await;
    });
    
    let client = OrderClient::new(tx);
    
    // Use client in handlers
    let app = Router::new()
        .route("/orders", post(handlers::create_order))
        .with_state(client);
    
    // ...
}
```

**Expected Outcome:**
- Thread-safe stateful actor with message-based API
- In-memory caching with automatic invalidation
- Decoupled business logic from HTTP layer
- Graceful shutdown via channel drop

---

## Pattern: Result-Based Error Handling

**Use case:** Explicit error propagation with `?` operator for clean async code.

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ServiceError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Not found: {resource} with id {id}")]
    NotFound { resource: String, id: String },
    
    #[error("Validation failed: {0}")]
    Validation(String),
}

pub async fn process_order(
    db: &PgPool,
    order_id: Uuid,
) -> Result<OrderResponse, ServiceError> {
    // ? operator propagates errors automatically
    let order = fetch_order(db, order_id).await?;
    let payment = process_payment(&order).await?;
    update_order_status(db, order_id, "completed").await?;
    
    Ok(OrderResponse {
        id: order.id,
        status: "completed".to_string(),
        payment_id: payment.id,
    })
}

async fn fetch_order(db: &PgPool, id: Uuid) -> Result<Order, ServiceError> {
    sqlx::query_as!(Order, "SELECT * FROM orders WHERE id = $1", id)
        .fetch_optional(db)
        .await?
        .ok_or(ServiceError::NotFound {
            resource: "Order".to_string(),
            id: id.to_string(),
        })
}
```

**Customization points:**
- Add context with `anyhow::Context` for richer error messages
- Implement custom `From` impls for third-party error types
- Use `eyre` for error reports with backtraces

---

## Pattern: Safe FFI Wrapper

**Use case:** Call C libraries with safe Rust interface.

```rust
use std::ffi::{CStr, CString};
use std::os::raw::{c_char, c_int};

// Unsafe FFI declarations
#[link(name = "z")]
extern "C" {
    fn compress(
        dest: *mut u8,
        dest_len: *mut usize,
        source: *const u8,
        source_len: usize,
    ) -> c_int;
}

// Safe wrapper
pub struct CompressionError {
    pub code: i32,
}

pub fn compress_data(data: &[u8]) -> Result<Vec<u8>, CompressionError> {
    let max_size = (data.len() as f64 * 1.1) as usize + 12; // zlib bound
    let mut compressed = vec![0u8; max_size];
    let mut compressed_size = max_size;
    
    let result = unsafe {
        compress(
            compressed.as_mut_ptr(),
            &mut compressed_size,
            data.as_ptr(),
            data.len(),
        )
    };
    
    if result != 0 {
        return Err(CompressionError { code: result });
    }
    
    compressed.truncate(compressed_size);
    Ok(compressed)
}

// Usage: fully safe API
let data = b"Hello, world!";
let compressed = compress_data(data).expect("Compression failed");
```

**Customization points:**
- Use `bindgen` to auto-generate FFI bindings from C headers
- Implement `Drop` for C resources (malloc'd pointers, file handles)
- Use `safer-ffi` for exporting Rust to C with memory safety guarantees

---

## Pattern: Trait-Based Dependency Injection

**Use case:** Testable code with mockable dependencies.

```rust
use async_trait::async_trait;
use uuid::Uuid;

#[async_trait]
pub trait OrderRepository: Send + Sync {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Order>, sqlx::Error>;
    async fn create(&self, order: &CreateOrder) -> Result<Order, sqlx::Error>;
}

// Production implementation
pub struct SqlxOrderRepository {
    pool: PgPool,
}

#[async_trait]
impl OrderRepository for SqlxOrderRepository {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Order>, sqlx::Error> {
        sqlx::query_as!(Order, "SELECT * FROM orders WHERE id = $1", id)
            .fetch_optional(&self.pool)
            .await
    }
    
    async fn create(&self, order: &CreateOrder) -> Result<Order, sqlx::Error> {
        // ... SQLx implementation
    }
}

// Mock for testing
pub struct MockOrderRepository {
    pub orders: HashMap<Uuid, Order>,
}

#[async_trait]
impl OrderRepository for MockOrderRepository {
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Order>, sqlx::Error> {
        Ok(self.orders.get(&id).cloned())
    }
    
    async fn create(&self, order: &CreateOrder) -> Result<Order, sqlx::Error> {
        // Mock implementation
    }
}

// Service uses trait
pub struct OrderService<R: OrderRepository> {
    repository: R,
}

impl<R: OrderRepository> OrderService<R> {
    pub async fn get_order(&self, id: Uuid) -> Result<Order, ServiceError> {
        self.repository
            .find_by_id(id)
            .await?
            .ok_or(ServiceError::NotFound {
                resource: "Order".to_string(),
                id: id.to_string(),
            })
    }
}

// Testing
#[tokio::test]
async fn test_get_order() {
    let mock_repo = MockOrderRepository {
        orders: HashMap::from([(order_id, order)]),
    };
    
    let service = OrderService { repository: mock_repo };
    let result = service.get_order(order_id).await.unwrap();
    
    assert_eq!(result.id, order_id);
}
```

**Customization points:**
- Use `mockall` crate for automatic mock generation
- Implement repository pattern for all external dependencies
- Use `Arc<dyn Trait>` for runtime polymorphism if compile-time generics not needed

---

## Axum Application Structure

```rust
// main.rs
use axum::{
    routing::{get, post},
    Router,
};
use sqlx::postgres::PgPoolOptions;
use std::sync::Arc;
use tower_http::trace::TraceLayer;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::init();
    
    let database_url = std::env::var("DATABASE_URL")?;
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await?;
    
    let app_state = Arc::new(AppState { db: pool });
    
    let app = Router::new()
        .route("/health", get(health_check))
        .route("/api/orders", post(create_order))
        .route("/api/orders/:id", get(get_order))
        .layer(TraceLayer::new_for_http())
        .with_state(app_state);
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    axum::serve(listener, app).await?;
    
    Ok(())
}

// handlers.rs
use axum::{
    extract::{Path, State},
    http::StatusCode,
    Json,
};

pub async fn create_order(
    State(state): State<Arc<AppState>>,
    Json(request): Json<CreateOrderRequest>,
) -> Result<(StatusCode, Json<OrderResponse>), AppError> {
    let order = service::create_order(&state.db, request).await?;
    Ok((StatusCode::CREATED, Json(order)))
}

pub async fn get_order(
    State(state): State<Arc<AppState>>,
    Path(id): Path<Uuid>,
) -> Result<Json<OrderResponse>, AppError> {
    let order = service::get_order(&state.db, id).await?;
    Ok(Json(order))
}
```

---

## Ownership Guidelines

| Pattern | Use When | Memory Behavior |
|---------|----------|-----------------|
| `T` (owned) | Transferring ownership | Caller loses access |
| `&T` (shared ref) | Read-only access | Zero-cost, no allocation |
| `&mut T` (mutable ref) | In-place modification | Exclusive access required |
| `Box<T>` | Heap allocation | Single owner |
| `Rc<T>` | Shared ownership (single-threaded) | Reference counted |
| `Arc<T>` | Shared ownership (multi-threaded) | Atomic reference counted |
| `Cow<'a, T>` | Clone-on-write | Avoid allocation when possible |
