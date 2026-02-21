# Networking & HTTP Patterns in Rust

Patterns for building networked applications — from raw TCP with tokio to HTTP servers with axum, HTTP clients with reqwest, middleware with tower, WebSockets, testing, and production hardening. Continues from Skills 1-150.

---

## Skill 151: Tokio I/O Fundamentals

**Pattern**: Tokio provides async versions of standard I/O traits. `TcpListener` accepts connections, `TcpStream` reads/writes, and `tokio::spawn` handles each connection concurrently.

```rust
use tokio::net::{TcpListener, TcpStream};
use tokio::io::{AsyncReadExt, AsyncWriteExt, AsyncBufReadExt, BufReader};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Listening on 127.0.0.1:8080");

    loop {
        let (stream, addr) = listener.accept().await?;
        println!("Connection from {addr}");
        tokio::spawn(async move {
            if let Err(e) = handle_connection(stream).await {
                eprintln!("Error handling {addr}: {e}");
            }
        });
    }
}

async fn handle_connection(stream: TcpStream) -> anyhow::Result<()> {
    let (reader, mut writer) = stream.into_split();
    let mut reader = BufReader::new(reader);
    let mut line = String::new();

    loop {
        line.clear();
        let n = reader.read_line(&mut line).await?;
        if n == 0 { break; } // EOF — client disconnected
        writer.write_all(line.as_bytes()).await?;
    }
    Ok(())
}
```

**Graceful shutdown** with `CancellationToken`:
```rust
use tokio_util::sync::CancellationToken;

async fn serve(token: CancellationToken) -> anyhow::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;

    loop {
        tokio::select! {
            result = listener.accept() => {
                let (stream, _) = result?;
                let child_token = token.child_token();
                tokio::spawn(async move {
                    handle_with_cancel(stream, child_token).await;
                });
            }
            _ = token.cancelled() => {
                println!("Shutting down");
                break;
            }
        }
    }
    Ok(())
}
```

**AsyncRead / AsyncWrite** — the core traits:
| Trait | Purpose |
|-------|---------|
| `AsyncRead` | `poll_read` — read bytes into a buffer |
| `AsyncWrite` | `poll_write`, `poll_flush`, `poll_shutdown` |
| `AsyncReadExt` | `.read()`, `.read_to_end()`, `.read_exact()` |
| `AsyncWriteExt` | `.write_all()`, `.write()`, `.flush()` |
| `AsyncBufReadExt` | `.read_line()`, `.lines()` |

**BufReader / BufWriter** — essential for performance:
```rust
use tokio::io::{BufReader, BufWriter};

let reader = BufReader::new(tcp_read_half);  // buffers reads
let writer = BufWriter::new(tcp_write_half); // buffers writes, flush on drop
```

**Skill to practice**: Build an echo server that accepts multiple concurrent connections. Add graceful shutdown with `CancellationToken` and Ctrl-C handling via `tokio::signal::ctrl_c()`.

---

## Skill 152: Framing & Codecs — tokio-util

**Pattern**: TCP is a byte stream, not a message stream. A codec splits the stream into frames (messages). `tokio-util` provides `Framed<T, Codec>` to adapt a stream into a `Sink + Stream` of frames.

```rust
use tokio_util::codec::{Framed, LinesCodec, LengthDelimitedCodec};
use futures::{StreamExt, SinkExt};

// Line-based protocol
async fn line_protocol(stream: TcpStream) -> anyhow::Result<()> {
    let mut framed = Framed::new(stream, LinesCodec::new());

    while let Some(line) = framed.next().await {
        let line = line?;
        println!("Got: {line}");
        framed.send(format!("echo: {line}")).await?;
    }
    Ok(())
}
```

**Built-in codecs**:
| Codec | Framing strategy |
|-------|-----------------|
| `LinesCodec` | Newline-delimited (`\n`) |
| `LengthDelimitedCodec` | 4-byte length prefix before each frame |
| `BytesCodec` | Raw bytes, no framing |

**Custom codec** — implement `Encoder` + `Decoder`:
```rust
use tokio_util::codec::{Decoder, Encoder};
use bytes::{BytesMut, Buf, BufMut};

struct JsonLineCodec;

impl Decoder for JsonLineCodec {
    type Item = serde_json::Value;
    type Error = anyhow::Error;

    fn decode(&mut self, src: &mut BytesMut) -> Result<Option<Self::Item>, Self::Error> {
        // Find newline
        let newline_pos = src.iter().position(|b| *b == b'\n');
        match newline_pos {
            Some(pos) => {
                let line = src.split_to(pos);
                src.advance(1); // skip the newline
                let value = serde_json::from_slice(&line)?;
                Ok(Some(value))
            }
            None => Ok(None), // Need more data
        }
    }
}

impl Encoder<serde_json::Value> for JsonLineCodec {
    type Error = anyhow::Error;

    fn encode(&mut self, item: serde_json::Value, dst: &mut BytesMut) -> Result<(), Self::Error> {
        let bytes = serde_json::to_vec(&item)?;
        dst.put_slice(&bytes);
        dst.put_u8(b'\n');
        Ok(())
    }
}
```

**Key rule in `Decoder::decode`**: Return `Ok(None)` when you don't have enough bytes yet. The framework will call you again after more data arrives. Never block waiting for data.

**Skill to practice**: Write a custom codec for a length-prefixed binary protocol (4-byte big-endian length + payload). Use it with `Framed` to send and receive messages.

---

## Skill 153: Tower — The Middleware Framework

**Pattern**: Tower defines `Service` — an async function from request to response. `Layer` wraps a service to add behavior (timeout, rate limiting, etc.). axum, hyper, and tonic all build on tower.

```rust
use tower::{Service, ServiceBuilder, ServiceExt};
use tower::timeout::TimeoutLayer;
use tower::limit::RateLimitLayer;
use tower::limit::ConcurrencyLimitLayer;
use std::time::Duration;

// The Service trait (simplified):
// trait Service<Request> {
//     type Response;
//     type Error;
//     type Future: Future<Output = Result<Response, Error>>;
//     fn poll_ready(&mut self, cx: &mut Context) -> Poll<Result<(), Error>>;
//     fn call(&mut self, req: Request) -> Self::Future;
// }

// Composing layers with ServiceBuilder
let service = ServiceBuilder::new()
    .layer(TimeoutLayer::new(Duration::from_secs(30)))
    .layer(RateLimitLayer::new(100, Duration::from_secs(1)))
    .layer(ConcurrencyLimitLayer::new(50))
    .service(my_service);

// ServiceBuilder applies layers bottom-to-top:
// Request → Timeout → RateLimit → ConcurrencyLimit → my_service
```

**Built-in layers**:
| Layer | Purpose |
|-------|---------|
| `TimeoutLayer` | Fail if service doesn't respond in time |
| `RateLimitLayer` | Max N requests per duration |
| `ConcurrencyLimitLayer` | Max N in-flight requests |
| `RetryLayer` | Retry failed requests (with policy) |
| `BufferLayer` | Add a channel buffer in front of a service |

**Writing a custom Layer**:
```rust
use tower::{Layer, Service};
use std::task::{Context, Poll};
use std::future::Future;
use std::pin::Pin;

#[derive(Clone)]
struct LogLayer;

impl<S> Layer<S> for LogLayer {
    type Service = LogService<S>;
    fn layer(&self, inner: S) -> Self::Service {
        LogService { inner }
    }
}

#[derive(Clone)]
struct LogService<S> { inner: S }

impl<S, Req> Service<Req> for LogService<S>
where
    S: Service<Req>,
    Req: std::fmt::Debug,
{
    type Response = S::Response;
    type Error = S::Error;
    type Future = S::Future;

    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        self.inner.poll_ready(cx)
    }

    fn call(&mut self, req: Req) -> Self::Future {
        println!("Request: {req:?}");
        self.inner.call(req)
    }
}
```

**Why tower matters**: Once you understand tower, you understand the middleware model for axum, tonic (gRPC), and hyper. A layer written for one works with all.

**Skill to practice**: Write a tower layer that measures request duration and prints it. Apply it to a simple service using `ServiceBuilder`.

---

## Skill 154: Axum Fundamentals — Routing & Handlers

**Pattern**: Axum routes map HTTP methods + paths to handler functions. Handlers are async functions that accept extractors and return `impl IntoResponse`.

```rust
use axum::{
    Router,
    routing::{get, post, put, delete},
    extract::{Path, Query, Json},
    response::IntoResponse,
    http::StatusCode,
};
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct Pagination {
    page: Option<u32>,
    per_page: Option<u32>,
}

#[derive(Serialize)]
struct User {
    id: u64,
    name: String,
}

// Simple handler — returns a string
async fn health() -> &'static str {
    "ok"
}

// Path extraction
async fn get_user(Path(id): Path<u64>) -> Json<User> {
    Json(User { id, name: "Alice".into() })
}

// Query extraction
async fn list_users(Query(params): Query<Pagination>) -> Json<Vec<User>> {
    let page = params.page.unwrap_or(1);
    let per_page = params.per_page.unwrap_or(20);
    // ... fetch from DB
    Json(vec![User { id: 1, name: "Alice".into() }])
}

// JSON body extraction
#[derive(Deserialize)]
struct CreateUser { name: String }

async fn create_user(Json(body): Json<CreateUser>) -> (StatusCode, Json<User>) {
    let user = User { id: 42, name: body.name };
    (StatusCode::CREATED, Json(user))
}

// Build the router
fn app() -> Router {
    Router::new()
        .route("/health", get(health))
        .route("/users", get(list_users).post(create_user))
        .route("/users/{id}", get(get_user))
}

#[tokio::main]
async fn main() {
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app()).await.unwrap();
}
```

**Nesting and fallback**:
```rust
let api = Router::new()
    .route("/users", get(list_users))
    .route("/users/{id}", get(get_user));

let app = Router::new()
    .nest("/api/v1", api)
    .fallback(|| async { (StatusCode::NOT_FOUND, "Not found") });
```

**Multiple path parameters**:
```rust
async fn get_post(
    Path((user_id, post_id)): Path<(u64, u64)>,
) -> String {
    format!("User {user_id}, Post {post_id}")
}

// .route("/users/{user_id}/posts/{post_id}", get(get_post))
```

**Skill to practice**: Build a CRUD API with routes for create, read, list, update, and delete. Use Path, Query, and Json extractors.

---

## Skill 155: Axum State & Dependency Injection

**Pattern**: Use `State` to inject shared application state (DB pool, config, etc.) into handlers. State must be `Clone + Send + Sync + 'static`.

```rust
use axum::{Router, routing::get, extract::State};
use sqlx::PgPool;
use std::sync::Arc;

// Simple state — wrap in Arc if Clone is expensive
#[derive(Clone)]
struct AppState {
    db: PgPool,
    config: Arc<AppConfig>,
}

struct AppConfig {
    jwt_secret: String,
    max_page_size: u32,
}

async fn list_users(State(state): State<AppState>) -> Json<Vec<User>> {
    let users = sqlx::query_as!(User, "SELECT id, name FROM users")
        .fetch_all(&state.db)
        .await
        .unwrap();
    Json(users)
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/users", get(list_users))
        .with_state(state)
}

#[tokio::main]
async fn main() {
    let pool = PgPool::connect("postgres://localhost/mydb").await.unwrap();
    let state = AppState {
        db: pool,
        config: Arc::new(AppConfig {
            jwt_secret: "secret".into(),
            max_page_size: 100,
        }),
    };
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app(state)).await.unwrap();
}
```

**Substates with `FromRef`** — extract a piece of the state:
```rust
use axum::extract::FromRef;

#[derive(Clone)]
struct AppState {
    db: PgPool,
    config: Arc<AppConfig>,
}

impl FromRef<AppState> for PgPool {
    fn from_ref(state: &AppState) -> Self {
        state.db.clone()
    }
}

// Now handlers can extract PgPool directly
async fn get_user(
    State(db): State<PgPool>,
    Path(id): Path<u64>,
) -> Json<User> {
    // use db directly
    todo!()
}
```

**State vs Extension**:
| | `State` | `Extension` |
|-|---------|-------------|
| Type safety | Compile-time (generic on Router) | Runtime (panics if missing) |
| When to use | App-wide shared state | Per-request values from middleware |
| Performance | Zero-cost, no HashMap lookup | HashMap lookup per extraction |

**Skill to practice**: Create an app with a database pool and config as shared state. Use `FromRef` to let handlers extract just the pool.

---

## Skill 156: Axum Middleware & Layers

**Pattern**: Axum uses tower layers for middleware. Apply with `Router::layer()`. Use `axum::middleware::from_fn` for quick custom middleware.

```rust
use axum::{
    Router, routing::get, middleware, extract::Request,
    response::Response, http::StatusCode,
};
use tower_http::{
    cors::CorsLayer,
    trace::TraceLayer,
    compression::CompressionLayer,
};

fn app() -> Router {
    Router::new()
        .route("/api/protected", get(protected_handler))
        .layer(middleware::from_fn(auth_middleware))
        .route("/health", get(health))
        // layers below apply to ALL routes above
        .layer(CompressionLayer::new())
        .layer(TraceLayer::new_for_http())
        .layer(CorsLayer::permissive())
}
```

**Custom middleware with `from_fn`**:
```rust
use axum::middleware::Next;

async fn auth_middleware(
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let auth_header = request.headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok());

    match auth_header {
        Some(token) if token.starts_with("Bearer ") => {
            // Token valid — continue
            Ok(next.run(request).await)
        }
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}
```

**Middleware with state**:
```rust
async fn auth_middleware(
    State(state): State<AppState>,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let token = request.headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let user = validate_jwt(token, &state.config.jwt_secret)
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    // Inject user into request extensions for handlers
    let mut request = request;
    request.extensions_mut().insert(user);
    Ok(next.run(request).await)
}

// Handler extracts the user from extensions
use axum::Extension;

async fn protected_handler(Extension(user): Extension<AuthUser>) -> String {
    format!("Hello, {}", user.name)
}
```

**Layer ordering** — layers wrap from bottom to top:
```rust
Router::new()
    .route("/", get(handler))
    .layer(A)  // outermost — runs first on request, last on response
    .layer(B)  // middle
    .layer(C)  // innermost — runs last on request, first on response
// Request flow:  A → B → C → handler
// Response flow: handler → C → B → A
```

**Common tower-http layers**:
| Layer | Purpose |
|-------|---------|
| `TraceLayer` | Request/response logging with tracing |
| `CorsLayer` | CORS headers |
| `CompressionLayer` | Gzip/brotli response compression |
| `TimeoutLayer` | Request timeout |
| `RequestBodyLimitLayer` | Limit request body size |

**Skill to practice**: Write an auth middleware that extracts a JWT from the Authorization header, validates it, and injects the user into request extensions.

---

## Skill 157: Axum Error Handling

**Pattern**: Define a custom error type that implements `IntoResponse`. Return `Result<T, AppError>` from handlers for consistent error responses.

```rust
use axum::{
    response::{IntoResponse, Response},
    http::StatusCode,
    Json,
};
use serde_json::json;

// Custom error type
enum AppError {
    NotFound(String),
    BadRequest(String),
    Internal(anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound(msg) => (StatusCode::NOT_FOUND, msg),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg),
            AppError::Internal(err) => {
                // Log the actual error, return generic message
                tracing::error!("Internal error: {err:?}");
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal server error".into())
            }
        };

        let body = Json(json!({
            "error": message,
        }));

        (status, body).into_response()
    }
}

// Implement From for automatic conversion with ?
impl From<sqlx::Error> for AppError {
    fn from(err: sqlx::Error) -> Self {
        match err {
            sqlx::Error::RowNotFound => AppError::NotFound("Resource not found".into()),
            _ => AppError::Internal(err.into()),
        }
    }
}

impl From<anyhow::Error> for AppError {
    fn from(err: anyhow::Error) -> Self {
        AppError::Internal(err)
    }
}

// Handlers return Result<T, AppError>
async fn get_user(
    State(db): State<PgPool>,
    Path(id): Path<u64>,
) -> Result<Json<User>, AppError> {
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id as i64)
        .fetch_one(&db)
        .await?; // sqlx::Error → AppError automatically
    Ok(Json(user))
}
```

**The anyhow + axum shortcut** (for prototyping):
```rust
// Quick and dirty — wraps any error into 500
struct AnyhowError(anyhow::Error);

impl IntoResponse for AnyhowError {
    fn into_response(self) -> Response {
        (StatusCode::INTERNAL_SERVER_ERROR, self.0.to_string()).into_response()
    }
}

impl<E: Into<anyhow::Error>> From<E> for AnyhowError {
    fn from(err: E) -> Self {
        AnyhowError(err.into())
    }
}

async fn handler() -> Result<String, AnyhowError> {
    let data = std::fs::read_to_string("file.txt")?;
    Ok(data)
}
```

**Consistent JSON error format**:
```rust
#[derive(Serialize)]
struct ErrorResponse {
    error: ErrorBody,
}

#[derive(Serialize)]
struct ErrorBody {
    code: &'static str,
    message: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    details: Option<serde_json::Value>,
}
```

**Skill to practice**: Create an `AppError` enum with variants for NotFound, Validation, Unauthorized, and Internal. Implement `IntoResponse` with JSON error bodies and proper status codes.

---

## Skill 158: Axum Extractors — Custom & Ordering

**Pattern**: Extractors pull data from requests. Implement `FromRequestParts` (for headers, query, path) or `FromRequest` (for body). Body-consuming extractors must be last in the handler signature.

```rust
use axum::{
    extract::{FromRequestParts, FromRequest, Request},
    http::{request::Parts, StatusCode, header},
    async_trait,
    response::{IntoResponse, Response},
    Json,
};

// Custom extractor: AuthenticatedUser
#[derive(Debug, Clone)]
struct AuthUser {
    user_id: u64,
    role: String,
}

// FromRequestParts — doesn't consume body
#[async_trait]
impl<S> FromRequestParts<S> for AuthUser
where
    S: Send + Sync,
{
    type Rejection = AuthError;

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let token = parts.headers
            .get(header::AUTHORIZATION)
            .and_then(|v| v.to_str().ok())
            .and_then(|v| v.strip_prefix("Bearer "))
            .ok_or(AuthError::MissingToken)?;

        // Validate token and extract claims
        let claims = decode_jwt(token)
            .map_err(|_| AuthError::InvalidToken)?;

        Ok(AuthUser {
            user_id: claims.sub,
            role: claims.role,
        })
    }
}

enum AuthError {
    MissingToken,
    InvalidToken,
}

impl IntoResponse for AuthError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            AuthError::MissingToken => (StatusCode::UNAUTHORIZED, "Missing token"),
            AuthError::InvalidToken => (StatusCode::UNAUTHORIZED, "Invalid token"),
        };
        (status, msg).into_response()
    }
}

// Use it in handlers — extracted before body
async fn create_post(
    user: AuthUser,              // FromRequestParts — extracted first
    Json(body): Json<CreatePost>, // FromRequest (body) — must be last
) -> Result<Json<Post>, AppError> {
    // user.user_id is available
    todo!()
}
```

**Extractor ordering rules**:
1. `FromRequestParts` extractors (Path, Query, State, Headers, custom) — any order
2. `FromRequest` extractors (Json, String, Bytes, Body) — **must be last**, only one

```rust
// ✅ Correct: body extractor last
async fn handler(Path(id): Path<u64>, Query(q): Query<Q>, Json(b): Json<B>) { }

// ❌ Wrong: Json is not last
// async fn handler(Json(b): Json<B>, Path(id): Path<u64>) { }
```

**Optional extractors**:
```rust
async fn handler(
    auth: Option<AuthUser>,  // None if extraction fails, no error returned
) -> String {
    match auth {
        Some(user) => format!("Hello, {}", user.user_id),
        None => "Hello, anonymous".into(),
    }
}
```

**Skill to practice**: Write a custom `FromRequestParts` extractor that reads an API key from a header and validates it against a list in the app state.

---

## Skill 159: reqwest — HTTP Client

**Pattern**: `reqwest::Client` is a connection-pooling HTTP client. Create one and reuse it across your application. Never create a `Client` per request.

```rust
use reqwest::Client;
use serde::{Deserialize, Serialize};
use std::time::Duration;

#[derive(Serialize)]
struct CreateTodo {
    title: String,
    completed: bool,
}

#[derive(Deserialize, Debug)]
struct Todo {
    id: u64,
    title: String,
    completed: bool,
}

// Create client once, reuse everywhere
fn build_client() -> reqwest::Result<Client> {
    Client::builder()
        .timeout(Duration::from_secs(30))
        .connect_timeout(Duration::from_secs(5))
        .pool_max_idle_per_host(10)
        .user_agent("my-app/1.0")
        .build()
}

async fn example(client: &Client) -> anyhow::Result<()> {
    // GET with query params
    let todos: Vec<Todo> = client
        .get("https://api.example.com/todos")
        .query(&[("page", "1"), ("limit", "10")])
        .header("Accept", "application/json")
        .send()
        .await?
        .error_for_status()?  // convert 4xx/5xx to Err
        .json()
        .await?;

    // POST with JSON body
    let new_todo = CreateTodo { title: "Learn Rust".into(), completed: false };
    let created: Todo = client
        .post("https://api.example.com/todos")
        .bearer_auth("my-token")
        .json(&new_todo)
        .send()
        .await?
        .error_for_status()?
        .json()
        .await?;

    // PUT
    client.put(format!("https://api.example.com/todos/{}", created.id))
        .json(&serde_json::json!({"completed": true}))
        .send()
        .await?
        .error_for_status()?;

    // DELETE
    client.delete(format!("https://api.example.com/todos/{}", created.id))
        .send()
        .await?
        .error_for_status()?;

    Ok(())
}
```

**Streaming large downloads**:
```rust
use tokio::io::AsyncWriteExt;
use futures::StreamExt;

async fn download_file(client: &Client, url: &str, path: &str) -> anyhow::Result<()> {
    let response = client.get(url).send().await?.error_for_status()?;
    let mut file = tokio::fs::File::create(path).await?;
    let mut stream = response.bytes_stream();

    while let Some(chunk) = stream.next().await {
        file.write_all(&chunk?).await?;
    }
    file.flush().await?;
    Ok(())
}
```

**Error handling pattern**:
```rust
async fn fetch_user(client: &Client, id: u64) -> Result<User, ApiError> {
    let response = client
        .get(format!("https://api.example.com/users/{id}"))
        .send()
        .await
        .map_err(|e| ApiError::Network(e.to_string()))?;

    match response.status() {
        StatusCode::OK => {
            let user = response.json().await
                .map_err(|e| ApiError::Deserialization(e.to_string()))?;
            Ok(user)
        }
        StatusCode::NOT_FOUND => Err(ApiError::NotFound(id)),
        status => {
            let body = response.text().await.unwrap_or_default();
            Err(ApiError::Server { status: status.as_u16(), body })
        }
    }
}
```

**Skill to practice**: Build a typed API client struct that wraps `reqwest::Client`, provides methods for each endpoint, and handles errors with a custom error type.

---

## Skill 160: WebSockets with axum

**Pattern**: Use `axum::extract::ws::WebSocketUpgrade` to upgrade HTTP connections to WebSockets. Split into sender/receiver for concurrent read/write.

```rust
use axum::{
    Router, routing::get,
    extract::ws::{WebSocket, WebSocketUpgrade, Message},
    response::IntoResponse,
};
use std::sync::Arc;
use tokio::sync::broadcast;

// Shared state for broadcasting
struct ChatState {
    tx: broadcast::Sender<String>,
}

async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<Arc<ChatState>>,
) -> impl IntoResponse {
    ws.on_upgrade(move |socket| handle_socket(socket, state))
}

async fn handle_socket(socket: WebSocket, state: Arc<ChatState>) {
    let (mut sender, mut receiver) = socket.split();
    let mut rx = state.tx.subscribe();

    // Task 1: forward broadcast messages to this client
    let mut send_task = tokio::spawn(async move {
        while let Ok(msg) = rx.recv().await {
            if sender.send(Message::Text(msg)).await.is_err() {
                break; // client disconnected
            }
        }
    });

    // Task 2: receive from this client and broadcast
    let tx = state.tx.clone();
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            if let Message::Text(text) = msg {
                let _ = tx.send(text);
            }
        }
    });

    // If either task exits, abort the other
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    }
}

fn app() -> Router<Arc<ChatState>> {
    Router::new().route("/ws", get(ws_handler))
}

#[tokio::main]
async fn main() {
    let (tx, _) = broadcast::channel(100);
    let state = Arc::new(ChatState { tx });
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app().with_state(state)).await.unwrap();
}
```

**Heartbeat / ping-pong**:
```rust
use tokio::time::{interval, Duration};

async fn handle_socket_with_heartbeat(mut socket: WebSocket) {
    let mut heartbeat = interval(Duration::from_secs(30));

    loop {
        tokio::select! {
            msg = socket.recv() => {
                match msg {
                    Some(Ok(Message::Pong(_))) => { /* client alive */ }
                    Some(Ok(Message::Text(text))) => {
                        // handle text message
                    }
                    Some(Ok(Message::Close(_))) | None => break,
                    _ => {}
                }
            }
            _ = heartbeat.tick() => {
                if socket.send(Message::Ping(vec![])).await.is_err() {
                    break; // client gone
                }
            }
        }
    }
}
```

**Skill to practice**: Build a chat server with WebSockets. Support multiple rooms using a `HashMap<String, broadcast::Sender<String>>` in shared state.

---

## Skill 161: HTTP Client Testing with wiremock

**Pattern**: `wiremock` starts a real HTTP server in tests. Mount expectations (method, path, headers), return canned responses, and verify that your client made the right requests.

```rust
use wiremock::{MockServer, Mock, ResponseTemplate};
use wiremock::matchers::{method, path, header, body_json};

#[tokio::test]
async fn test_get_user() {
    let server = MockServer::start().await;

    Mock::given(method("GET"))
        .and(path("/users/42"))
        .and(header("Authorization", "Bearer test-token"))
        .respond_with(
            ResponseTemplate::new(200)
                .set_body_json(serde_json::json!({
                    "id": 42,
                    "name": "Alice"
                }))
        )
        .expect(1)  // verify exactly 1 request
        .mount(&server)
        .await;

    // Point your client at the mock server
    let client = ApiClient::new(&server.uri(), "test-token");
    let user = client.get_user(42).await.unwrap();

    assert_eq!(user.name, "Alice");
    // expectations verified on MockServer drop
}
```

**Sequenced responses** (different response per call):
```rust
use wiremock::matchers::any;

#[tokio::test]
async fn test_retry_on_failure() {
    let server = MockServer::start().await;

    // First call: 500
    Mock::given(any())
        .respond_with(ResponseTemplate::new(500))
        .up_to_n_times(1)
        .expect(1)
        .mount(&server).await;

    // Second call: 200
    Mock::given(any())
        .respond_with(ResponseTemplate::new(200).set_body_json(json!({"ok": true})))
        .expect(1)
        .mount(&server).await;

    let client = RetryingClient::new(&server.uri());
    let result = client.fetch().await.unwrap();
    assert_eq!(result["ok"], true);
}
```

**Testing error scenarios**:
```rust
#[tokio::test]
async fn test_timeout_handling() {
    let server = MockServer::start().await;

    Mock::given(any())
        .respond_with(
            ResponseTemplate::new(200)
                .set_body_string("slow")
                .set_delay(Duration::from_secs(10)) // simulate slow server
        )
        .mount(&server).await;

    let client = Client::builder()
        .timeout(Duration::from_millis(100))
        .build().unwrap();

    let result = client.get(format!("{}/data", server.uri()))
        .send().await;

    assert!(result.is_err()); // timeout
}

#[tokio::test]
async fn test_404_handling() {
    let server = MockServer::start().await;

    Mock::given(method("GET"))
        .and(path("/users/999"))
        .respond_with(ResponseTemplate::new(404))
        .mount(&server).await;

    let client = ApiClient::new(&server.uri(), "token");
    let err = client.get_user(999).await.unwrap_err();
    assert!(matches!(err, ApiError::NotFound(999)));
}
```

**Verifying request bodies**:
```rust
#[tokio::test]
async fn test_create_sends_correct_body() {
    let server = MockServer::start().await;

    Mock::given(method("POST"))
        .and(path("/users"))
        .and(body_json(serde_json::json!({
            "name": "Bob",
            "email": "bob@example.com"
        })))
        .respond_with(ResponseTemplate::new(201))
        .expect(1)
        .mount(&server).await;

    let client = ApiClient::new(&server.uri(), "token");
    client.create_user("Bob", "bob@example.com").await.unwrap();
}
```

**Skill to practice**: Write wiremock tests for an API client covering: success, 404, 500, timeout, and request body verification.

---

## Skill 162: Connection Pooling & Timeouts

**Pattern**: Reuse connections across requests. Apply timeouts at every level. Use retry with backoff for transient failures.

**reqwest connection pooling** — create once, reuse always:
```rust
use reqwest::Client;
use std::time::Duration;

// ✅ Create once at startup
let client = Client::builder()
    .pool_idle_timeout(Duration::from_secs(90))
    .pool_max_idle_per_host(10)
    .connect_timeout(Duration::from_secs(5))
    .timeout(Duration::from_secs(30))
    .build()?;

// ❌ Don't create per request
async fn bad_fetch(url: &str) -> reqwest::Result<String> {
    reqwest::get(url).await?.text().await  // new client each time!
}
```

**Database connection pooling**:
```rust
use sqlx::postgres::PgPoolOptions;

let pool = PgPoolOptions::new()
    .max_connections(20)
    .min_connections(5)
    .acquire_timeout(Duration::from_secs(5))
    .idle_timeout(Duration::from_secs(600))
    .max_lifetime(Duration::from_secs(1800))
    .connect("postgres://localhost/mydb")
    .await?;
```

**Tower timeout and rate limit layers**:
```rust
use tower::ServiceBuilder;
use tower::timeout::TimeoutLayer;
use tower::limit::RateLimitLayer;

let service = ServiceBuilder::new()
    .layer(TimeoutLayer::new(Duration::from_secs(10)))
    .layer(RateLimitLayer::new(100, Duration::from_secs(1)))
    .service(inner_service);
```

**Retry with exponential backoff**:
```rust
use std::time::Duration;
use tokio::time::sleep;

async fn retry_with_backoff<F, Fut, T, E>(
    mut f: F,
    max_retries: u32,
) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: std::future::Future<Output = Result<T, E>>,
{
    let mut attempt = 0;
    loop {
        match f().await {
            Ok(val) => return Ok(val),
            Err(e) if attempt < max_retries => {
                attempt += 1;
                let delay = Duration::from_millis(100 * 2u64.pow(attempt));
                let jitter = Duration::from_millis(rand::random::<u64>() % 100);
                sleep(delay + jitter).await;
            }
            Err(e) => return Err(e),
        }
    }
}
```

**Circuit breaker pattern** (simplified):
```rust
use std::sync::atomic::{AtomicU32, AtomicBool, Ordering};
use std::sync::Arc;

struct CircuitBreaker {
    failure_count: AtomicU32,
    is_open: AtomicBool,
    threshold: u32,
}

impl CircuitBreaker {
    fn new(threshold: u32) -> Self {
        Self {
            failure_count: AtomicU32::new(0),
            is_open: AtomicBool::new(false),
            threshold,
        }
    }

    fn record_success(&self) {
        self.failure_count.store(0, Ordering::Relaxed);
        self.is_open.store(false, Ordering::Relaxed);
    }

    fn record_failure(&self) {
        let count = self.failure_count.fetch_add(1, Ordering::Relaxed) + 1;
        if count >= self.threshold {
            self.is_open.store(true, Ordering::Relaxed);
        }
    }

    fn is_open(&self) -> bool {
        self.is_open.load(Ordering::Relaxed)
    }
}
```

**Skill to practice**: Build an HTTP client wrapper with connection pooling, configurable timeouts, and retry with exponential backoff.

---

## Skill 163: TLS & Security

**Pattern**: Use `rustls` (pure Rust) for TLS. Configure custom certificates for internal services. Apply security headers via middleware.

**rustls vs native-tls**:
| | `rustls` | `native-tls` |
|-|----------|-------------|
| Implementation | Pure Rust | Wraps OS library (OpenSSL, SChannel, Security.framework) |
| Default for reqwest | `features = ["rustls-tls"]` | `features = ["native-tls"]` |
| Certificate store | `webpki-roots` or system | System |
| Recommended | Yes — auditable, no C deps | When OS cert store is required |

**reqwest with custom certificates**:
```rust
use reqwest::Certificate;

let cert = std::fs::read("internal-ca.pem")?;
let cert = Certificate::from_pem(&cert)?;

let client = Client::builder()
    .add_root_certificate(cert)
    .build()?;
```

**reqwest with client certificates** (mTLS):
```rust
use reqwest::Identity;

let cert = std::fs::read("client-cert.pem")?;
let key = std::fs::read("client-key.pem")?;
let identity = Identity::from_pem(&[cert, key].concat())?;

let client = Client::builder()
    .identity(identity)
    .build()?;
```

**Axum with TLS** (using `axum-server`):
```rust
use axum_server::tls_rustls::RustlsConfig;

#[tokio::main]
async fn main() {
    let config = RustlsConfig::from_pem_file("cert.pem", "key.pem")
        .await
        .unwrap();

    let app = Router::new().route("/", get(|| async { "Hello, TLS!" }));

    axum_server::bind_rustls("0.0.0.0:443".parse().unwrap(), config)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

**Security headers middleware**:
```rust
use axum::{middleware, extract::Request, response::Response};
use axum::http::HeaderValue;

async fn security_headers(request: Request, next: middleware::Next) -> Response {
    let mut response = next.run(request).await;
    let headers = response.headers_mut();

    headers.insert("X-Content-Type-Options", HeaderValue::from_static("nosniff"));
    headers.insert("X-Frame-Options", HeaderValue::from_static("DENY"));
    headers.insert("X-XSS-Protection", HeaderValue::from_static("1; mode=block"));
    headers.insert(
        "Strict-Transport-Security",
        HeaderValue::from_static("max-age=31536000; includeSubDomains"),
    );
    headers.insert(
        "Content-Security-Policy",
        HeaderValue::from_static("default-src 'self'"),
    );

    response
}
```

**Best practice**: In production, prefer a reverse proxy (nginx, Caddy) for TLS termination. Use axum-server TLS for development or when a reverse proxy isn't available.

**Skill to practice**: Configure a reqwest client with a custom CA certificate and connect to a local TLS server. Add security headers middleware to an axum app.

---

## Skill 164: Networking Decision Matrix & Best Practices

**Pattern**: Choose the right framework, apply production best practices, and avoid common pitfalls.

**Framework comparison**:
| | axum | actix-web | warp |
|-|------|-----------|------|
| Tower compatible | Yes (built on tower) | No (own middleware) | Partial |
| Extractors | Type-safe, composable | Similar, good ergonomics | Filter-based |
| Ecosystem | Growing fast, Tokio team | Mature, large ecosystem | Smaller community |
| Performance | Excellent | Excellent | Excellent |
| Recommendation | **Default choice** | Large existing codebase | Existing warp projects |

**Graceful shutdown**:
```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "ok" }));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    let ctrl_c = signal::ctrl_c();
    let mut sigterm = signal::unix::signal(signal::unix::SignalKind::terminate()).unwrap();

    tokio::select! {
        _ = ctrl_c => println!("Ctrl-C received"),
        _ = sigterm.recv() => println!("SIGTERM received"),
    }
}
```

**Health checks and readiness probes**:
```rust
async fn health() -> &'static str { "ok" }

async fn ready(State(db): State<PgPool>) -> StatusCode {
    match sqlx::query("SELECT 1").execute(&db).await {
        Ok(_) => StatusCode::OK,
        Err(_) => StatusCode::SERVICE_UNAVAILABLE,
    }
}

let app = Router::new()
    .route("/health", get(health))     // liveness — is the process alive?
    .route("/ready", get(ready));      // readiness — can it serve traffic?
```

**Structured logging with tracing**:
```rust
use tower_http::trace::TraceLayer;
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

#[tokio::main]
async fn main() {
    tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer().json())
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    let app = Router::new()
        .route("/", get(handler))
        .layer(TraceLayer::new_for_http());

    // Every request is logged with method, path, status, latency
}
```

**Don't do**:
| Anti-pattern | Why it's bad | Fix |
|-------------|-------------|-----|
| Blocking in async handlers | Starves the tokio runtime | Use `tokio::task::spawn_blocking` |
| `reqwest::Client` per request | No connection reuse, socket exhaustion | Create once, share via State |
| Unbounded `tokio::spawn` | Memory/task explosion under load | Use semaphore or concurrency limit |
| Ignoring backpressure | OOM under load | Bounded channels, body size limits |
| `unwrap()` in handlers | 500 on any unexpected input | Return `Result<T, AppError>` |
| Logging secrets | Security breach | Redact auth headers in middleware |

**Production checklist**:
1. Graceful shutdown (drain connections)
2. Health + readiness endpoints
3. Request timeouts (tower `TimeoutLayer`)
4. Body size limits (`RequestBodyLimitLayer`)
5. CORS configuration
6. Structured JSON logging
7. Metrics (prometheus, metrics crate)
8. Connection pooling for DB and HTTP clients
9. TLS (direct or via reverse proxy)
10. Error responses: never leak internal details

**Skill to practice**: Set up a production-ready axum server with graceful shutdown, health checks, structured logging, CORS, timeout, and body size limits.

---

## Quick Reference

| Skill | Pattern | One-Liner |
|-------|---------|-----------|
| 151 | Tokio I/O Fundamentals | TcpListener, TcpStream, spawn per connection, graceful shutdown |
| 152 | Framing & Codecs | `Framed<T, Codec>`, LinesCodec, custom Encoder/Decoder |
| 153 | Tower Middleware | `Service` + `Layer` traits, ServiceBuilder, composable middleware |
| 154 | Axum Routing & Handlers | Router, Path/Query/Json extractors, IntoResponse |
| 155 | Axum State & DI | `State(pool)`, `with_state()`, `FromRef` for substates |
| 156 | Axum Middleware | `Router::layer()`, `from_fn`, tower-http layers, ordering |
| 157 | Axum Error Handling | Custom error → IntoResponse, Result<T, AppError>, JSON errors |
| 158 | Custom Extractors | `FromRequestParts` / `FromRequest`, ordering rules, rejections |
| 159 | reqwest HTTP Client | Reuse Client, builder pattern, error_for_status, streaming |
| 160 | WebSockets | WebSocketUpgrade, split sender/receiver, broadcast, heartbeat |
| 161 | wiremock Testing | MockServer, matchers, sequenced responses, verify requests |
| 162 | Pooling & Timeouts | Connection pools, retry with backoff, circuit breaker |
| 163 | TLS & Security | rustls, custom certs, mTLS, security headers |
| 164 | Decision Matrix | axum default, graceful shutdown, health checks, anti-patterns |

---

## Change Log

| Date | Change |
|------|--------|
| 2026-02-18 | Initial version — Skills 151-164 from tokio/axum/tower/reqwest analysis |
