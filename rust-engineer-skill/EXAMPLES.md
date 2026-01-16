# Rust Engineer - Code Examples & Patterns

This document contains code examples, anti-patterns, and real-world implementation patterns for Rust development.

## Anti-Pattern 1: Ignoring Error Handling with .unwrap()

**What it looks like:**
```rust
// WRONG: Unwrap everywhere
fn load_config() -> Config {
    let content = std::fs::read_to_string("config.toml").unwrap(); // Panic on missing file!
    toml::from_str(&content).unwrap() // Panic on parse error!
}

// WRONG: Ignoring Results
async fn save_user(db: &PgPool, user: &User) {
    sqlx::query!("INSERT INTO users (name) VALUES ($1)", user.name)
        .execute(db)
        .await; // Result ignored! Error silently discarded
}
```

**Why it fails:**
- **Runtime panics**: `.unwrap()` crashes the program on `Err` or `None`
- **Silent failures**: Ignoring `Result` hides database errors, network failures
- **No recovery**: Panics can't be caught safely in async contexts
- **Poor UX**: Users see "thread panicked" instead of helpful error messages

**Correct approach:**
```rust
// CORRECT: Proper error propagation
fn load_config() -> Result<Config, ConfigError> {
    let content = std::fs::read_to_string("config.toml")
        .map_err(|e| ConfigError::FileRead(e))?;
    toml::from_str(&content)
        .map_err(|e| ConfigError::Parse(e))
}

// CORRECT: Handle errors explicitly
async fn save_user(db: &PgPool, user: &User) -> Result<(), sqlx::Error> {
    sqlx::query!("INSERT INTO users (name) VALUES ($1)", user.name)
        .execute(db)
        .await?;
    Ok(())
}

// CORRECT: Use expect() only for truly impossible cases
let port: u16 = std::env::var("PORT")
    .expect("PORT env var required") // Only if this is a startup requirement
    .parse()
    .expect("PORT must be a valid u16");
```

**When .unwrap() is acceptable:**
- Tests (failure should fail the test)
- Prototypes and examples
- Truly impossible cases (mathematically proven)
- After explicit validation (`if let Some(x) = ... { x.unwrap() }`)

---

## Anti-Pattern 2: Clone-Heavy Code (Unnecessary Allocations)

**What it looks like:**
```rust
// WRONG: Cloning on every function call
fn process_orders(orders: Vec<Order>) -> Vec<OrderSummary> {
    orders
        .iter()
        .map(|order| calculate_summary(order.clone())) // Unnecessary clone!
        .collect()
}

// WRONG: Cloning to satisfy borrow checker
async fn handle_request(state: AppState) {
    let db = state.db.clone(); // Clone entire connection pool!
    let config = state.config.clone(); // Clone large config!
    
    process_data(db, config).await;
}

// WRONG: String clones in hot path
fn generate_ids(count: usize) -> Vec<String> {
    (0..count)
        .map(|i| format!("ID-{}", i).clone()) // format! already returns String!
        .collect()
}
```

**Why it fails:**
- **Performance degradation**: Allocations are expensive
- **Memory pressure**: Unnecessary clones increase memory usage
- **Cache misses**: More allocations = more pointer chasing
- **Hides ownership bugs**: Cloning masks design issues

**Correct approach:**
```rust
// CORRECT: Borrow instead of clone
fn process_orders(orders: &[Order]) -> Vec<OrderSummary> {
    orders
        .iter()
        .map(|order| calculate_summary(order)) // Pass reference
        .collect()
}

fn calculate_summary(order: &Order) -> OrderSummary {
    OrderSummary {
        id: order.id,
        total: order.items.iter().map(|i| i.price).sum(),
    }
}

// CORRECT: Share with Arc only when needed
#[derive(Clone)]
struct AppState {
    db: PgPool, // PgPool is already Arc internally, cheap to clone
    config: Arc<Config>, // Wrap large struct in Arc
}

async fn handle_request(state: AppState) {
    // Cloning state is cheap: PgPool = Arc clone, config = Arc clone
    process_data(state.db, state.config).await;
}

// CORRECT: Avoid redundant operations
fn generate_ids(count: usize) -> Vec<String> {
    (0..count)
        .map(|i| format!("ID-{}", i)) // format! returns String, no .clone()
        .collect()
}

// CORRECT: Use Cow for conditional cloning
use std::borrow::Cow;

fn normalize_string(s: &str) -> Cow<str> {
    if s.contains(char::is_uppercase) {
        Cow::Owned(s.to_lowercase()) // Clone only if needed
    } else {
        Cow::Borrowed(s) // No allocation if already lowercase
    }
}
```

**Guidelines:**
- **Default to borrowing** (`&T`) unless ownership transfer needed
- **Use `Arc<T>`** for shared immutable data across threads
- **Use `Rc<T>`** for shared data in single-threaded context
- **Profile before optimizing**: Use `cargo flamegraph` to find hot clones
- **Cheap clones are OK**: `Arc`, `Rc`, `PgPool`, small Copy types

---

## Example: Complete Axum REST API

```rust
// main.rs
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::IntoResponse,
    routing::{delete, get, post, put},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use sqlx::postgres::PgPoolOptions;
use std::sync::Arc;
use uuid::Uuid;

#[derive(Clone)]
struct AppState {
    db: sqlx::PgPool,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let database_url = std::env::var("DATABASE_URL")?;
    
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&database_url)
        .await?;
    
    let state = AppState { db: pool };
    
    let app = Router::new()
        .route("/api/products", get(list_products).post(create_product))
        .route(
            "/api/products/:id",
            get(get_product).put(update_product).delete(delete_product),
        )
        .with_state(state);
    
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    println!("Server running on http://localhost:3000");
    axum::serve(listener, app).await?;
    
    Ok(())
}

// Models
#[derive(Debug, Serialize, Deserialize, sqlx::FromRow)]
struct Product {
    id: Uuid,
    name: String,
    price: f64,
    created_at: chrono::DateTime<chrono::Utc>,
}

#[derive(Debug, Deserialize)]
struct CreateProduct {
    name: String,
    price: f64,
}

#[derive(Debug, Deserialize)]
struct ListParams {
    limit: Option<i64>,
    offset: Option<i64>,
}

// Handlers
async fn list_products(
    State(state): State<AppState>,
    Query(params): Query<ListParams>,
) -> Result<Json<Vec<Product>>, AppError> {
    let limit = params.limit.unwrap_or(20);
    let offset = params.offset.unwrap_or(0);
    
    let products = sqlx::query_as!(
        Product,
        r#"SELECT id, name, price, created_at FROM products 
           ORDER BY created_at DESC LIMIT $1 OFFSET $2"#,
        limit,
        offset
    )
    .fetch_all(&state.db)
    .await?;
    
    Ok(Json(products))
}

async fn get_product(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> Result<Json<Product>, AppError> {
    let product = sqlx::query_as!(
        Product,
        "SELECT id, name, price, created_at FROM products WHERE id = $1",
        id
    )
    .fetch_optional(&state.db)
    .await?
    .ok_or(AppError::NotFound)?;
    
    Ok(Json(product))
}

async fn create_product(
    State(state): State<AppState>,
    Json(input): Json<CreateProduct>,
) -> Result<(StatusCode, Json<Product>), AppError> {
    let product = sqlx::query_as!(
        Product,
        r#"INSERT INTO products (id, name, price, created_at) 
           VALUES ($1, $2, $3, NOW()) 
           RETURNING id, name, price, created_at"#,
        Uuid::new_v4(),
        input.name,
        input.price
    )
    .fetch_one(&state.db)
    .await?;
    
    Ok((StatusCode::CREATED, Json(product)))
}

// Error handling
#[derive(Debug)]
enum AppError {
    Database(sqlx::Error),
    NotFound,
}

impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        AppError::Database(err)
    }
}

impl IntoResponse for AppError {
    fn into_response(self) -> axum::response::Response {
        let (status, message) = match self {
            AppError::Database(e) => {
                tracing::error!("Database error: {:?}", e);
                (StatusCode::INTERNAL_SERVER_ERROR, "Database error")
            }
            AppError::NotFound => (StatusCode::NOT_FOUND, "Not found"),
        };
        
        (status, Json(serde_json::json!({ "error": message }))).into_response()
    }
}
```

---

## Testing Patterns

### Unit Testing with Mocks

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[tokio::test]
    async fn test_create_order_success() {
        let mock_repo = MockOrderRepository::new();
        mock_repo.expect_create()
            .returning(|order| Ok(Order { id: Uuid::new_v4(), ..order }));
        
        let service = OrderService::new(mock_repo);
        let result = service.create(CreateOrder { 
            customer_id: Uuid::new_v4(),
            items: vec![],
        }).await;
        
        assert!(result.is_ok());
    }
    
    #[tokio::test]
    async fn test_get_order_not_found() {
        let mock_repo = MockOrderRepository::new();
        mock_repo.expect_find_by_id()
            .returning(|_| Ok(None));
        
        let service = OrderService::new(mock_repo);
        let result = service.get(Uuid::new_v4()).await;
        
        assert!(matches!(result, Err(ServiceError::NotFound { .. })));
    }
}
```

### Integration Testing

```rust
#[cfg(test)]
mod integration_tests {
    use super::*;
    use sqlx::PgPool;
    
    #[sqlx::test]
    async fn test_product_crud(pool: PgPool) {
        // Create
        let product = create_product(&pool, "Test Product", 10.0).await.unwrap();
        assert_eq!(product.name, "Test Product");
        
        // Read
        let fetched = get_product(&pool, product.id).await.unwrap();
        assert_eq!(fetched.id, product.id);
        
        // Update
        let updated = update_product(&pool, product.id, "Updated", 20.0).await.unwrap();
        assert_eq!(updated.name, "Updated");
        assert_eq!(updated.price, 20.0);
        
        // Delete
        delete_product(&pool, product.id).await.unwrap();
        let result = get_product(&pool, product.id).await;
        assert!(result.is_err());
    }
}
```

---

## Dockerfile for Rust Services

```dockerfile
# Build stage
FROM rust:1.75-alpine AS builder

RUN apk add --no-cache musl-dev

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src

RUN cargo build --release --target x86_64-unknown-linux-musl

# Runtime stage
FROM alpine:3.19

RUN apk add --no-cache ca-certificates

COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /usr/local/bin/

EXPOSE 3000
CMD ["myapp"]
```

**Build optimizations:**
- Use `lto = true` in `Cargo.toml` for smaller binaries
- Use `strip = true` to remove debug symbols
- Consider `opt-level = "s"` for size optimization
