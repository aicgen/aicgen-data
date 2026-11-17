# Rust Web Services And CLIs

Recommended building blocks (check current versions before adding):

| Need | Crate |
|------|-------|
| HTTP server | `axum` (on `tokio`, `hyper`, `tower`) |
| HTTP client | `reqwest` |
| Serialization | `serde`, `serde_json` |
| CLI arguments | `clap` (derive API) |
| Logging and tracing | `tracing`, `tracing-subscriber` |
| Database | `sqlx` (async, compile-time checked) or `diesel` |
| Errors | `thiserror` (libraries), `anyhow` (binaries) |
| Config | `figment`, `config`, or plain env vars + `serde` |
| Secrets in memory | `secrecy` |

## Service Layout

```
myservice/
├── Cargo.toml
├── migrations/             # sqlx migrations
└── src/
    ├── main.rs             # load config, init tracing, build state, serve
    ├── lib.rs              # pub fn app(state) -> Router, for tests
    ├── config.rs
    ├── error.rs            # ApiError + IntoResponse
    ├── state.rs            # AppState: DB pool, clients, settings
    └── users/
        ├── mod.rs
        ├── handlers.rs     # extract -> call service -> map to response
        ├── service.rs      # business rules, no HTTP types
        └── repo.rs         # SQL
```

- Handlers stay thin: extract, validate, call a service function, convert errors. Business logic never touches HTTP types.
- `lib.rs` exposes `fn app(state: AppState) -> Router` so integration tests can call it without binding a port.

## An axum Handler

```rust,ignore
use axum::{
    Json, Router,
    extract::{Path, State},
    http::StatusCode,
    response::{IntoResponse, Response},
    routing::{get, post},
};
use serde::{Deserialize, Serialize};

#[derive(Clone)]
pub struct AppState {
    pub users: UserService,
}

#[derive(Deserialize)]
struct CreateUser {
    email: String,
}

#[derive(Serialize)]
struct UserDto {
    id: u64,
    email: String,
}

// Path parameters use `{name}` in axum 0.8+ (older versions used `:name`)
pub fn app(state: AppState) -> Router {
    Router::new()
        .route("/users", post(create_user))
        .route("/users/{id}", get(get_user))
        .with_state(state)
}

async fn create_user(
    State(state): State<AppState>,
    Json(body): Json<CreateUser>,
) -> Result<(StatusCode, Json<UserDto>), ApiError> {
    let email = Email::parse(&body.email).map_err(|_| ApiError::Validation("invalid email"))?;
    let user = state.users.create(email).await?;
    Ok((StatusCode::CREATED, Json(UserDto { id: user.id, email: user.email.to_string() })))
}

async fn get_user(State(state): State<AppState>, Path(id): Path<u64>) -> Result<Json<UserDto>, ApiError> {
    let user = state.users.find(id).await?.ok_or(ApiError::NotFound)?;
    Ok(Json(UserDto { id: user.id, email: user.email.to_string() }))
}

// One error type converted to a stable HTTP response; never leak internals
#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("not found")]
    NotFound,
    #[error("{0}")]
    Validation(&'static str),
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            Self::NotFound => (StatusCode::NOT_FOUND, "not found".to_owned()),
            Self::Validation(msg) => (StatusCode::UNPROCESSABLE_ENTITY, (*msg).to_owned()),
            Self::Internal(err) => {
                tracing::error!(error = ?err, "request failed");
                (StatusCode::INTERNAL_SERVER_ERROR, "internal error".to_owned())
            }
        };
        (status, Json(serde_json::json!({ "error": message }))).into_response()
    }
}
```

- Parse into domain types (`Email`, `UserId`) at the edge; services receive already-valid values.
- Use separate request/response DTOs from database and domain models so internal changes do not break the API contract.
- Set `#[serde(deny_unknown_fields)]` on request types when strictness matters, and `#[serde(rename_all = "camelCase")]` to match the wire format.

## Middleware (tower layers)

```rust,ignore
use std::time::Duration;
use tower_http::{limit::RequestBodyLimitLayer, timeout::TimeoutLayer, trace::TraceLayer};

let app = app(state)
    .layer(TraceLayer::new_for_http())                       // request spans
    .layer(RequestBodyLimitLayer::new(1024 * 1024))          // cap request bodies
    .layer(TimeoutLayer::new(Duration::from_secs(10)));      // no unbounded handlers
```

- Always set body-size limits and request timeouts. Add rate limiting (`tower_governor`) and CORS (`tower_http::cors`) with an explicit allow-list for browser clients.
- Authentication belongs in a layer or an extractor, not copy-pasted into handlers.

## Startup And Graceful Shutdown

```rust,ignore
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env()) // RUST_LOG=info
        .init();

    let settings = Settings::load()?;
    let pool = sqlx::PgPool::connect(settings.database_url.expose_secret()).await?;
    sqlx::migrate!().run(&pool).await?;

    let listener = tokio::net::TcpListener::bind(settings.addr).await?;
    tracing::info!(addr = %settings.addr, "listening");
    axum::serve(listener, app(AppState::new(pool)))
        .with_graceful_shutdown(shutdown_signal())
        .await?;
    Ok(())
}

async fn shutdown_signal() {
    let ctrl_c = async { tokio::signal::ctrl_c().await.expect("install Ctrl+C handler") };
    #[cfg(unix)]
    let terminate = async {
        tokio::signal::unix::signal(tokio::signal::unix::SignalKind::terminate())
            .expect("install SIGTERM handler")
            .recv()
            .await;
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! { _ = ctrl_c => {}, _ = terminate => {} }
}
```

- Expose `/healthz` (liveness) and `/readyz` (checks dependencies) endpoints.
- Log structured fields (`tracing::info!(user_id = %id, "created")`), not formatted strings. Propagate a request/trace id.

## Database (sqlx)

```rust,ignore
// ✅ Bind parameters; never format values into SQL
let user = sqlx::query_as!(UserRow, "SELECT id, email FROM users WHERE id = $1", id)
    .fetch_optional(&pool)
    .await?;

// ✅ Transactions for multi-step writes
let mut tx = pool.begin().await?;
sqlx::query!("UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from)
    .execute(&mut *tx)
    .await?;
sqlx::query!("UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to)
    .execute(&mut *tx)
    .await?;
tx.commit().await?;
```

- Use the macros' compile-time checking (`cargo sqlx prepare` for offline CI builds) or typed query builders.
- Size the pool for the database's connection limit and set acquire timeouts.
- Keep migrations in version control and run them as a deploy step, not implicitly on every instance if you scale horizontally.

## Configuration And Secrets

```rust,ignore
use secrecy::{ExposeSecret, SecretString};
use serde::Deserialize;

#[derive(Deserialize)]
pub struct Settings {
    pub addr: std::net::SocketAddr,
    pub database_url: SecretString,   // Debug prints "[REDACTED]"
}
```

- Layer configuration: defaults in code, optional file, then environment variables. Fail fast at startup with a clear message when required values are missing.
- Never commit secrets, never log them, and never include them in error messages or `Debug` output (use `secrecy`).

## CLI Applications (clap)

```rust,ignore
use clap::{Parser, Subcommand};
use std::process::ExitCode;

#[derive(Parser)]
#[command(version, about = "Manage widgets")]
struct Cli {
    /// Increase log verbosity (-v, -vv)
    #[arg(short, long, action = clap::ArgAction::Count, global = true)]
    verbose: u8,

    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand)]
enum Command {
    /// Create a widget
    Create { name: String },
    /// List widgets
    List {
        #[arg(long, default_value_t = 20)]
        limit: usize,
    },
}

fn main() -> ExitCode {
    let cli = Cli::parse();
    match run(cli) {
        Ok(()) => ExitCode::SUCCESS,
        Err(err) => {
            eprintln!("error: {err:#}");   // anyhow chain on one line, to stderr
            ExitCode::FAILURE
        }
    }
}
```

- Write results to **stdout** and diagnostics to **stderr** so output can be piped.
- Return a non-zero exit code on failure; use distinct codes only when scripts will rely on them.
- Offer `--json` for machine-readable output. Keep human formatting separate from data.
- Respect `NO_COLOR`, detect TTYs (`std::io::IsTerminal`), and avoid interactive prompts when stdin is not a terminal.
- Test with `assert_cmd` and `predicates`; test pure logic as library functions.

## Quick Checklist

- [ ] Handlers are thin; domain logic has no HTTP or SQL types
- [ ] Input is parsed into validated domain types at the boundary
- [ ] Body limits, timeouts and graceful shutdown are configured
- [ ] SQL uses bound parameters; secrets use `secrecy` and are never logged
- [ ] One `ApiError` maps internal errors to stable, non-leaky responses
- [ ] CLI separates stdout/stderr and returns meaningful exit codes
