# CLI Patterns in Rust

Patterns for building robust command-line applications — argument parsing with clap, layered configuration, structured logging with tracing, progress indicators, signal handling, testing, and best practices. Continues from Skills 1-164.

---

## Skill 165: clap Fundamentals — Derive API

**Pattern**: Use `#[derive(Parser)]` to define CLI arguments as a struct. clap generates parsing, help text, and validation from your type definitions.

```rust
use clap::Parser;
use std::path::PathBuf;

/// A tool for processing data files
#[derive(Parser, Debug)]
#[command(version, about, long_about = None)]
struct Cli {
    /// Input file to process
    input: PathBuf,

    /// Output file (defaults to stdout)
    #[arg(short, long)]
    output: Option<PathBuf>,

    /// Number of worker threads
    #[arg(short = 'j', long, default_value_t = 4)]
    threads: usize,

    /// Enable verbose output
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,

    /// API key (can also be set via env var)
    #[arg(long, env = "MY_TOOL_API_KEY")]
    api_key: Option<String>,
}

fn main() {
    let cli = Cli::parse();
    println!("{cli:?}");
}
```

**How it works**:
- Positional args are fields without `#[arg(short, long)]` — order matters
- `#[arg(short)]` gives `-o`, `#[arg(long)]` gives `--output`, both gives `-o`/`--output`
- `Option<T>` makes an arg optional, bare `T` makes it required
- `#[arg(default_value_t = 4)]` provides a default (type must impl `Display`)
- `#[arg(env = "VAR")]` falls back to an environment variable
- Doc comments (`///`) become the help text — both for the command and each arg
- `#[command(version)]` pulls the version from `Cargo.toml`

**Cargo.toml**:
```toml
[dependencies]
clap = { version = "4", features = ["derive", "env"] }
```

**Common arg types**:
| Rust Type | CLI Behavior |
|-----------|-------------|
| `String` | Required string arg |
| `Option<String>` | Optional string arg |
| `PathBuf` | Path arg (no validation by default) |
| `bool` | Flag — `--flag` sets to true |
| `u8` with `ArgAction::Count` | `-vvv` counts to 3 |
| `Vec<String>` | Repeatable — `--tag a --tag b` |

**Skill to practice**: Define a CLI struct with a required positional arg, two optional flags, an env-backed argument, and a count flag for verbosity. Run with `--help` to see the generated help text.

---

## Skill 166: clap Subcommands & Enums

**Pattern**: Use `#[derive(Subcommand)]` on an enum to define subcommands. Each variant is a subcommand with its own arguments.

```rust
use clap::{Parser, Subcommand, Args};

#[derive(Parser)]
#[command(version, about = "A project management tool")]
struct Cli {
    /// Enable debug logging
    #[arg(long, global = true)]
    debug: bool,

    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Initialize a new project
    Init {
        /// Project name
        name: String,
        /// Use a template
        #[arg(short, long)]
        template: Option<String>,
    },

    /// Run the project
    Run(RunArgs),

    /// Manage configuration
    #[command(subcommand)]
    Config(ConfigCommands),
}

/// Arguments for the run subcommand
#[derive(Args)]
struct RunArgs {
    /// Port to listen on
    #[arg(short, long, default_value_t = 8080)]
    port: u16,

    /// Enable hot reload
    #[arg(long)]
    watch: bool,
}

#[derive(Subcommand)]
enum ConfigCommands {
    /// Set a config value
    Set {
        key: String,
        value: String,
    },
    /// Get a config value
    Get {
        key: String,
    },
    /// List all config values
    List,
}

fn main() {
    let cli = Cli::parse();

    // Global args are accessible at the top level
    if cli.debug {
        eprintln!("Debug mode enabled");
    }

    match cli.command {
        Commands::Init { name, template } => {
            println!("Creating project '{name}'");
            if let Some(t) = template {
                println!("Using template: {t}");
            }
        }
        Commands::Run(args) => {
            println!("Running on port {}", args.port);
        }
        Commands::Config(cmd) => match cmd {
            ConfigCommands::Set { key, value } => println!("Setting {key}={value}"),
            ConfigCommands::Get { key } => println!("Getting {key}"),
            ConfigCommands::List => println!("Listing all config"),
        },
    }
}
```

**Shared option groups** with `#[command(flatten)]`:
```rust
#[derive(Args)]
struct DatabaseOpts {
    /// Database URL
    #[arg(long, env = "DATABASE_URL")]
    database_url: String,

    /// Max connections in pool
    #[arg(long, default_value_t = 10)]
    max_connections: u32,
}

#[derive(Subcommand)]
enum Commands {
    Migrate(MigrateArgs),
    Serve(ServeArgs),
}

#[derive(Args)]
struct MigrateArgs {
    #[command(flatten)]
    db: DatabaseOpts, // shared DB options appear in both subcommands
}

#[derive(Args)]
struct ServeArgs {
    #[command(flatten)]
    db: DatabaseOpts,

    #[arg(short, long, default_value_t = 3000)]
    port: u16,
}
```

**Usage**:
```bash
mycli init my-project --template web
mycli run --port 3000 --watch
mycli config set theme dark
mycli --debug run                  # global flag before subcommand
```

**Skill to practice**: Build a CLI with three subcommands, one of which has nested subcommands. Use `#[command(flatten)]` to share an option group between two subcommands.

---

## Skill 167: clap Validation & Value Types

**Pattern**: Use custom value parsers, `ValueEnum`, and argument groups to validate inputs at parse time rather than after.

**Enum args with `ValueEnum`**:
```rust
use clap::{Parser, ValueEnum};

#[derive(Debug, Clone, ValueEnum)]
enum OutputFormat {
    Json,
    Yaml,
    Toml,
    #[value(alias = "txt")]
    Plain,
}

#[derive(Parser)]
struct Cli {
    /// Output format
    #[arg(short, long, default_value_t = OutputFormat::Json)]
    format: OutputFormat,
}

// ValueEnum requires Display for default_value_t
impl std::fmt::Display for OutputFormat {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::Json => write!(f, "json"),
            Self::Yaml => write!(f, "yaml"),
            Self::Toml => write!(f, "toml"),
            Self::Plain => write!(f, "plain"),
        }
    }
}
```

**Custom value parsers**:
```rust
use clap::Parser;
use std::path::PathBuf;

fn parse_existing_path(s: &str) -> Result<PathBuf, String> {
    let path = PathBuf::from(s);
    if path.exists() {
        Ok(path)
    } else {
        Err(format!("path does not exist: {s}"))
    }
}

fn parse_port_range(s: &str) -> Result<u16, String> {
    let port: u16 = s.parse().map_err(|_| format!("not a valid port: {s}"))?;
    if (1024..=65535).contains(&port) {
        Ok(port)
    } else {
        Err(format!("port must be between 1024 and 65535, got {port}"))
    }
}

#[derive(Parser)]
struct Cli {
    /// Input file (must exist)
    #[arg(value_parser = parse_existing_path)]
    input: PathBuf,

    /// Port number (1024-65535)
    #[arg(short, long, value_parser = parse_port_range)]
    port: u16,
}
```

**Mutually exclusive args and groups**:
```rust
use clap::{Parser, ArgGroup};

#[derive(Parser)]
#[command(group(ArgGroup::new("source")
    .required(true)
    .args(["file", "url"])))]
struct Cli {
    /// Read from file
    #[arg(long)]
    file: Option<PathBuf>,

    /// Read from URL
    #[arg(long)]
    url: Option<String>,

    /// Required with --url
    #[arg(long, requires = "url")]
    auth_token: Option<String>,
}
```

**`num_args` for controlling value counts**:
```rust
#[derive(Parser)]
struct Cli {
    /// Exactly two coordinates
    #[arg(long, num_args = 2)]
    point: Vec<f64>,

    /// One to three tags
    #[arg(long, num_args = 1..=3)]
    tags: Vec<String>,
}
```

**Built-in value parsers**: clap provides `value_parser!(PathBuf)`, `value_parser!(u16).range(1024..=65535)`, and others for common validations without writing custom functions.

**Skill to practice**: Create a CLI with a `ValueEnum` for output format, a custom value parser that validates a path exists, and a mutually exclusive argument group.

---

## Skill 168: Configuration Management — Layered Config

**Pattern**: Merge configuration from multiple sources with clear precedence: defaults < config file < environment variables < CLI arguments. The `config` crate (config-rs) handles this elegantly.

```rust
use config::{Config, ConfigError, Environment, File};
use serde::Deserialize;
use std::path::PathBuf;

#[derive(Debug, Deserialize)]
pub struct AppConfig {
    pub host: String,
    pub port: u16,
    pub database_url: String,
    pub log_level: String,
    pub max_connections: u32,
    pub feature_flags: FeatureFlags,
}

#[derive(Debug, Deserialize)]
pub struct FeatureFlags {
    pub new_ui: bool,
    pub beta_api: bool,
}

impl AppConfig {
    pub fn load(config_path: Option<&PathBuf>, cli_overrides: &CliArgs) -> Result<Self, ConfigError> {
        let mut builder = Config::builder()
            // Layer 1: Hardcoded defaults
            .set_default("host", "127.0.0.1")?
            .set_default("port", 8080)?
            .set_default("log_level", "info")?
            .set_default("max_connections", 10)?
            .set_default("feature_flags.new_ui", false)?
            .set_default("feature_flags.beta_api", false)?;

        // Layer 2: Config file (optional)
        if let Some(path) = config_path {
            builder = builder.add_source(File::from(path.as_ref()));
        } else {
            // Look for config.toml in standard locations
            builder = builder.add_source(
                File::with_name("config").required(false)
            );
        }

        // Layer 3: Environment variables (prefix: MYAPP_)
        // MYAPP_PORT=3000 → port = 3000
        // MYAPP_DATABASE_URL=... → database_url = ...
        // MYAPP_FEATURE_FLAGS__NEW_UI=true → feature_flags.new_ui = true
        builder = builder.add_source(
            Environment::with_prefix("MYAPP")
                .separator("__")
                .try_parsing(true)
        );

        // Layer 4: CLI overrides (highest precedence)
        if let Some(port) = cli_overrides.port {
            builder = builder.set_override("port", port as i64)?;
        }
        if let Some(ref log_level) = cli_overrides.log_level {
            builder = builder.set_override("log_level", log_level.as_str())?;
        }

        builder.build()?.try_deserialize()
    }
}
```

**Config file (`config.toml`)**:
```toml
host = "0.0.0.0"
port = 3000
database_url = "postgres://localhost/mydb"
log_level = "debug"

[feature_flags]
new_ui = true
```

**Using `dotenvy` for `.env` files** (development convenience):
```rust
fn main() {
    // Load .env file if present (before config loading)
    dotenvy::dotenv().ok(); // .ok() ignores missing file

    let config = AppConfig::load(None, &cli_args).unwrap();
}
```

```
# .env
MYAPP_DATABASE_URL=postgres://localhost/mydb_dev
MYAPP_LOG_LEVEL=debug
```

**The `figment` crate** — alternative with provider composition:
```rust
use figment::{Figment, providers::{Format, Toml, Env, Serialized}};

let config: AppConfig = Figment::new()
    .merge(Serialized::defaults(AppConfig::default()))
    .merge(Toml::file("config.toml"))
    .merge(Env::prefixed("MYAPP_").split("__"))
    .extract()?;
```

**Precedence summary**:
| Priority | Source | Example |
|:--------:|--------|---------|
| 1 (lowest) | Hardcoded defaults | `set_default("port", 8080)` |
| 2 | Config file | `config.toml` |
| 3 | `.env` file (via dotenvy) | `.env` |
| 4 | Environment variables | `MYAPP_PORT=3000` |
| 5 (highest) | CLI arguments | `--port 3000` |

**Skill to practice**: Build a config loader that merges defaults, a TOML file, environment variables, and CLI overrides. Verify that CLI args take precedence over everything.

---

## Skill 169: tracing — Structured Logging

**Pattern**: Use `tracing` instead of `log`/`env_logger` for structured, contextual logging. Events carry typed key-value fields, not just formatted strings.

```rust
use tracing::{info, warn, error, debug, trace, instrument};

// Structured fields — machine-parseable
fn handle_request(user_id: u64, path: &str) {
    info!(user_id, path, "handling request");
    // Output: 2026-02-20T10:00:00Z INFO handle_request: handling request user_id=42 path="/api/users"

    // Display vs Debug formatting
    let user = lookup_user(user_id);
    info!(user_id, user_name = %user.name, "found user");     // %  → Display
    debug!(user_id, user_details = ?user, "user details");     // ?  → Debug

    // Conditional fields
    if user.is_admin {
        warn!(user_id, role = "admin", "admin access");
    }
}

// #[instrument] creates a span around the function
#[instrument(skip(password), fields(user_email = %email))]
async fn login(email: &str, password: &str) -> Result<Token, AuthError> {
    info!("attempting login");  // automatically includes email span

    let user = find_user(email).await?;
    let token = create_token(&user)?;

    info!(token_expiry = ?token.expires_at, "login successful");
    Ok(token)
}
// All events inside login() will show: login{user_email=alice@ex.com}: ...
```

**Setting up the subscriber**:
```rust
use tracing_subscriber::{fmt, EnvFilter};

fn init_logging() {
    // Simple setup — respects RUST_LOG env var
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info"))
        )
        .with_target(true)       // show module path
        .with_thread_ids(true)   // show thread IDs
        .with_file(true)         // show source file
        .with_line_number(true)  // show line number
        .init();
}

fn main() {
    init_logging();
    info!("application starting");
}
```

**Filtering with `RUST_LOG`**:
```bash
RUST_LOG=info                          # global info level
RUST_LOG=my_crate=debug                # debug for my crate, default for others
RUST_LOG=my_crate::db=trace,warn       # trace for db module, warn for everything else
RUST_LOG=my_crate[login]=debug         # debug only inside 'login' spans
```

**Spans for tracking request lifecycle**:
```rust
use tracing::{info_span, Instrument};

async fn handle(req: Request) -> Response {
    let span = info_span!("request",
        method = %req.method(),
        path = %req.uri().path(),
        request_id = %Uuid::new_v4(),
    );

    async move {
        info!("processing");  // inherits request span fields
        let response = process(req).await;
        info!(status = %response.status(), "completed");
        response
    }
    .instrument(span)
    .await
}
```

**tracing vs log/env_logger**:
| Feature | log/env_logger | tracing |
|---------|---------------|---------|
| Structured fields | No (format strings only) | Yes (typed key-value) |
| Spans (context) | No | Yes |
| Async-aware | No | Yes |
| Multiple outputs | Manual | Layer composition |
| Performance | Good | Excellent (compile-time filtering) |

**Skill to practice**: Replace `println!` debugging with `tracing` in a project. Use `#[instrument]` on key functions and query with `RUST_LOG` filters.

---

## Skill 170: tracing Layers & Subscribers

**Pattern**: Compose multiple outputs (console, file, JSON, telemetry) using `tracing_subscriber`'s layer system. Each layer independently processes events.

```rust
use tracing_subscriber::{fmt, Registry, EnvFilter, Layer};
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::util::SubscriberInitExt;

fn init_tracing() {
    // Console layer — human-readable, filtered
    let console_layer = fmt::layer()
        .with_target(true)
        .with_filter(EnvFilter::new("info"));

    // JSON layer — for log aggregation (e.g., shipped to ELK/Loki)
    let json_layer = fmt::layer()
        .json()
        .with_current_span(true)
        .with_span_list(true)
        .with_filter(EnvFilter::new("debug"));

    // File layer — with rolling files
    let file_appender = tracing_appender::rolling::daily("/var/log/myapp", "app.log");
    let (non_blocking, _guard) = tracing_appender::non_blocking(file_appender);
    // IMPORTANT: _guard must be held — dropping it loses buffered logs
    let file_layer = fmt::layer()
        .with_writer(non_blocking)
        .with_ansi(false)  // no colors in files
        .with_filter(EnvFilter::new("debug"));

    // Compose all layers on a single Registry
    Registry::default()
        .with(console_layer)
        .with(json_layer)
        .with(file_layer)
        .init();
}
```

**Hold the guard**: The non-blocking writer returns a guard. If you drop it, buffered logs are lost. Store it in `main()`:
```rust
fn main() {
    let (_file_guard, _) = init_tracing();
    // _file_guard lives until main() returns
    run_app();
}
```

**JSON output format** (one event per line):
```json
{"timestamp":"2026-02-20T10:00:00Z","level":"INFO","fields":{"message":"handling request","user_id":42,"path":"/api/users"},"target":"my_crate::server","span":{"name":"request","request_id":"abc-123"}}
```

**Custom layers** — implement the `Layer` trait:
```rust
use tracing_subscriber::Layer;
use tracing::Subscriber;

struct MetricsLayer;

impl<S: Subscriber> Layer<S> for MetricsLayer {
    fn on_event(&self, event: &tracing::Event<'_>, _ctx: tracing_subscriber::layer::Context<'_, S>) {
        if *event.metadata().level() == tracing::Level::ERROR {
            // Increment error counter in your metrics system
            metrics::counter!("errors_total").increment(1);
        }
    }
}
```

**OpenTelemetry integration** for distributed tracing:
```rust
use tracing_opentelemetry::OpenTelemetryLayer;
use opentelemetry::trace::TracerProvider;
use opentelemetry_otlp::WithExportConfig;

fn init_otel() -> impl tracing::Subscriber {
    let exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .with_endpoint("http://localhost:4317")
        .build()
        .unwrap();

    let provider = opentelemetry_sdk::trace::TracerProvider::builder()
        .with_batch_exporter(exporter)
        .build();

    let tracer = provider.tracer("my-service");

    Registry::default()
        .with(OpenTelemetryLayer::new(tracer))
        .with(fmt::layer())
}
```

**Cargo.toml for tracing ecosystem**:
```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
tracing-appender = "0.2"
# Optional: OpenTelemetry
tracing-opentelemetry = "0.22"
opentelemetry = "0.21"
opentelemetry-otlp = "0.14"
```

**Skill to practice**: Set up a subscriber with three layers — console (info+), JSON file (debug+), and a custom layer that counts errors. Verify each output independently.

---

## Skill 171: Progress & User Interaction

**Pattern**: Use `indicatif` for progress bars, `dialoguer` for prompts, and `console` for styling. Always write UI to stderr so stdout stays clean for data.

**Progress bars with `indicatif`**:
```rust
use indicatif::{ProgressBar, ProgressStyle, MultiProgress, HumanDuration};
use std::time::Duration;

// Simple progress bar
fn download_files(urls: &[&str]) {
    let pb = ProgressBar::new(urls.len() as u64);
    pb.set_style(ProgressStyle::default_bar()
        .template("{spinner:.green} [{elapsed_precise}] [{bar:40.cyan/blue}] {pos}/{len} ({eta})")
        .unwrap()
        .progress_chars("#>-"));

    for url in urls {
        pb.set_message(format!("Downloading {url}"));
        // ... actual download ...
        pb.inc(1);
    }
    pb.finish_with_message("All downloads complete");
}

// Spinner for indeterminate operations
fn compile_project() {
    let spinner = ProgressBar::new_spinner();
    spinner.set_style(ProgressStyle::default_spinner()
        .template("{spinner:.green} {msg}")
        .unwrap());
    spinner.set_message("Compiling...");
    spinner.enable_steady_tick(Duration::from_millis(100));

    // ... do work ...

    spinner.finish_with_message("Compilation complete");
}

// Multiple concurrent progress bars
fn parallel_downloads(urls: &[&str]) {
    let multi = MultiProgress::new();

    let handles: Vec<_> = urls.iter().map(|url| {
        let pb = multi.add(ProgressBar::new(100));
        pb.set_style(ProgressStyle::default_bar()
            .template("{msg:30} [{bar:20}] {pos}%")
            .unwrap());
        pb.set_message(url.to_string());

        std::thread::spawn(move || {
            for i in 0..100 {
                pb.set_position(i);
                std::thread::sleep(Duration::from_millis(50));
            }
            pb.finish_with_message("done");
        })
    }).collect();

    for h in handles { h.join().unwrap(); }
}
```

**Interactive prompts with `dialoguer`**:
```rust
use dialoguer::{Confirm, Input, Select, MultiSelect, Password, theme::ColorfulTheme};

fn setup_wizard() -> Result<(), Box<dyn std::error::Error>> {
    let project_name: String = Input::with_theme(&ColorfulTheme::default())
        .with_prompt("Project name")
        .default("my-project".into())
        .interact_text()?;

    let framework = Select::with_theme(&ColorfulTheme::default())
        .with_prompt("Choose a framework")
        .items(&["Axum", "Actix-web", "Rocket", "Warp"])
        .default(0)
        .interact()?;

    let features = MultiSelect::with_theme(&ColorfulTheme::default())
        .with_prompt("Select features")
        .items(&["Database", "Auth", "WebSockets", "Metrics"])
        .interact()?;

    let proceed = Confirm::with_theme(&ColorfulTheme::default())
        .with_prompt("Create project?")
        .default(true)
        .interact()?;

    if proceed {
        println!("Creating {project_name}...");
    }

    Ok(())
}
```

**Styling with `console`**:
```rust
use console::{style, Emoji};

static SUCCESS: Emoji = Emoji("✅ ", "OK ");
static WARN: Emoji = Emoji("⚠️  ", "WARN ");

fn report(ok: bool) {
    if ok {
        eprintln!("{} {}", SUCCESS, style("Build succeeded").green().bold());
    } else {
        eprintln!("{} {}", WARN, style("Build failed").red().bold());
    }
}
```

**Stderr vs stdout rule**: Data goes to stdout (so users can pipe it). UI, progress, and prompts go to stderr.
```rust
// Correct
eprintln!("Processing...");        // UI → stderr
println!("{}", json_output);       // data → stdout

// indicatif writes to stderr by default — correct behavior
```

**Skill to practice**: Build a CLI with a multi-step setup wizard using `dialoguer`, and display a progress bar while processing each step using `indicatif`.

---

## Skill 172: Signal Handling & Graceful Shutdown

**Pattern**: Catch signals (Ctrl+C, SIGTERM) and shut down cleanly — stop accepting work, drain in-flight tasks, then exit.

**Async graceful shutdown with tokio**:
```rust
use tokio::signal;
use tokio::sync::watch;
use tokio_util::sync::CancellationToken;

async fn run_server() {
    let token = CancellationToken::new();

    // Spawn the shutdown listener
    let shutdown_token = token.clone();
    tokio::spawn(async move {
        shutdown_signal().await;
        tracing::info!("shutdown signal received");
        shutdown_token.cancel();
    });

    // Main server loop
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    tracing::info!("listening on :3000");

    loop {
        tokio::select! {
            Ok((stream, addr)) = listener.accept() => {
                let token = token.clone();
                tokio::spawn(async move {
                    handle_connection(stream, addr, token).await;
                });
            }
            _ = token.cancelled() => {
                tracing::info!("stopping listener");
                break;
            }
        }
    }

    // Drain in-flight connections (CancellationToken propagates)
    tracing::info!("waiting for in-flight requests to complete...");
    // In practice, use a WaitGroup or track task handles
    tokio::time::sleep(std::time::Duration::from_secs(5)).await;
    tracing::info!("shutdown complete");
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to listen for ctrl+c");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to listen for SIGTERM")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

**Using `CancellationToken` for cooperative shutdown**:
```rust
use tokio_util::sync::CancellationToken;

async fn handle_connection(
    stream: TcpStream,
    addr: SocketAddr,
    token: CancellationToken,
) {
    loop {
        tokio::select! {
            result = read_request(&stream) => {
                match result {
                    Ok(req) => process(req).await,
                    Err(_) => break,
                }
            }
            _ = token.cancelled() => {
                tracing::info!(%addr, "connection shutting down");
                send_goodbye(&stream).await.ok();
                break;
            }
        }
    }
}
```

**Sync signal handling with `ctrlc` crate**:
```rust
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::Arc;

fn main() {
    let running = Arc::new(AtomicBool::new(true));
    let r = running.clone();

    ctrlc::set_handler(move || {
        eprintln!("\nShutting down...");
        r.store(false, Ordering::SeqCst);
    }).expect("Error setting Ctrl-C handler");

    while running.load(Ordering::SeqCst) {
        // do work
    }

    cleanup();
}
```

**Cleanup on exit** — temp files, lock files, PID files:
```rust
struct LockFile { path: PathBuf }

impl LockFile {
    fn acquire(path: PathBuf) -> std::io::Result<Self> {
        std::fs::write(&path, std::process::id().to_string())?;
        Ok(Self { path })
    }
}

impl Drop for LockFile {
    fn drop(&mut self) {
        let _ = std::fs::remove_file(&self.path);
    }
}
// Lock file removed even on panic (Drop runs during unwinding)
```

**Skill to practice**: Build an async server that handles Ctrl+C and SIGTERM, stops accepting connections, waits up to 10 seconds for in-flight work to drain, then exits.

---

## Skill 173: Testing CLI Applications

**Pattern**: Test CLI binaries end-to-end with `assert_cmd`, use `trycmd` for snapshot-based testing, and inject configuration/stdin for interactive tests.

**`assert_cmd` for binary testing**:
```rust
use assert_cmd::Command;
use predicates::prelude::*;

#[test]
fn help_flag_works() {
    Command::cargo_bin("my-tool").unwrap()
        .arg("--help")
        .assert()
        .success()
        .stdout(predicate::str::contains("Usage:"))
        .stdout(predicate::str::contains("--output"));
}

#[test]
fn missing_required_arg_fails() {
    Command::cargo_bin("my-tool").unwrap()
        .assert()
        .failure()
        .stderr(predicate::str::contains("required"))
        .code(2);  // clap uses exit code 2 for usage errors
}

#[test]
fn processes_input_file() {
    let dir = tempfile::tempdir().unwrap();
    let input = dir.path().join("input.txt");
    std::fs::write(&input, "hello world").unwrap();

    Command::cargo_bin("my-tool").unwrap()
        .arg(&input)
        .arg("--format").arg("json")
        .assert()
        .success()
        .stdout(predicate::str::contains("\"hello world\""));
}

#[test]
fn stdin_input() {
    Command::cargo_bin("my-tool").unwrap()
        .arg("--from-stdin")
        .write_stdin("line1\nline2\n")
        .assert()
        .success()
        .stdout(predicate::str::contains("2 lines"));
}
```

**Environment variables in tests**:
```rust
#[test]
fn respects_env_config() {
    Command::cargo_bin("my-tool").unwrap()
        .env("MYAPP_LOG_LEVEL", "debug")
        .env("MYAPP_PORT", "9090")
        .arg("config").arg("show")
        .assert()
        .success()
        .stdout(predicate::str::contains("port: 9090"));
}
```

**`trycmd` for snapshot-based CLI testing**:
```
// tests/cmd/help.md
```console
$ my-tool --help
A tool for processing data files

Usage: my-tool [OPTIONS] <INPUT>

Arguments:
  <INPUT>  Input file to process

Options:
  -o, --output <OUTPUT>  Output file
  -h, --help             Print help
  -V, --version          Print version
```
```

```rust
// tests/cli_tests.rs
#[test]
fn cli_tests() {
    trycmd::TestCases::new()
        .case("tests/cmd/*.md");
}
```

**Testing config file loading**:
```rust
#[test]
fn loads_config_from_file() {
    let dir = tempfile::tempdir().unwrap();
    let config_path = dir.path().join("config.toml");
    std::fs::write(&config_path, r#"
        port = 9090
        log_level = "debug"
    "#).unwrap();

    Command::cargo_bin("my-tool").unwrap()
        .arg("--config").arg(&config_path)
        .arg("show-config")
        .assert()
        .success()
        .stdout(predicate::str::contains("port: 9090"));
}
```

**Integration test organization for CLI apps**:
```
tests/
  cli/
    mod.rs              # shared helpers
    test_init.rs        # tests for `init` subcommand
    test_run.rs         # tests for `run` subcommand
    test_config.rs      # tests for `config` subcommand
  cmd/
    help.md             # trycmd snapshots
    init.md
    errors.md
  integration.rs        # imports cli/ modules
```

**Skill to practice**: Write assert_cmd tests for a CLI covering: help output, missing args, valid input, invalid input, stdin piping, and environment variable overrides.

---

## Skill 174: CLI Best Practices & Anti-Patterns

**Pattern**: Follow conventions that make CLI tools composable, scriptable, and user-friendly.

**Exit codes**:
```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,           // 0
        Err(e) if e.is_user_error() => {
            eprintln!("Error: {e}");
            ExitCode::from(1)                  // 1 = runtime error
        }
        Err(e) => {
            eprintln!("Error: {e}");
            ExitCode::from(2)                  // 2 = usage error
        }
    }
}
```

**`anyhow` for clean error reporting in main()**:
```rust
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let config_path = std::env::args()
        .nth(1)
        .context("usage: my-tool <config-file>")?;

    let config = std::fs::read_to_string(&config_path)
        .with_context(|| format!("failed to read config file: {config_path}"))?;

    let parsed: Config = toml::from_str(&config)
        .context("failed to parse config")?;

    run(parsed)?;
    Ok(())
}
// Output on error:
// Error: failed to read config file: missing.toml
//
// Caused by:
//     No such file or directory (os error 2)
```

**Respecting `--quiet` and `--verbose`**:
```rust
#[derive(Parser)]
struct Cli {
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,

    #[arg(short, long)]
    quiet: bool,
}

fn init_logging(cli: &Cli) {
    let level = if cli.quiet {
        "error"
    } else {
        match cli.verbose {
            0 => "warn",
            1 => "info",
            2 => "debug",
            _ => "trace",
        }
    };
    tracing_subscriber::fmt()
        .with_env_filter(level)
        .with_writer(std::io::stderr)  // logs to stderr
        .init();
}
```

**Shell completions with `clap_complete`**:
```rust
use clap::{CommandFactory, Parser};
use clap_complete::{generate, Shell};

#[derive(Parser)]
struct Cli {
    /// Generate shell completions
    #[arg(long, value_enum)]
    completions: Option<Shell>,

    // ... other args
}

fn main() {
    let cli = Cli::parse();

    if let Some(shell) = cli.completions {
        generate(shell, &mut Cli::command(), "my-tool", &mut std::io::stdout());
        return;
    }

    // ... normal execution
}
```
```bash
# Generate and install completions
my-tool --completions bash > ~/.local/share/bash-completion/completions/my-tool
my-tool --completions zsh > ~/.zfunc/_my-tool
my-tool --completions fish > ~/.config/fish/completions/my-tool.fish
```

**Man page generation**:
```rust
// build.rs
use clap::CommandFactory;
use clap_mangen::Man;

fn main() -> std::io::Result<()> {
    let out_dir = std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap());
    let cmd = Cli::command();
    let man = Man::new(cmd);
    let mut buffer = Vec::new();
    man.render(&mut buffer)?;
    std::fs::write(out_dir.join("my-tool.1"), buffer)?;
    Ok(())
}
```

**Anti-patterns to avoid**:
| Anti-Pattern | Why It's Bad | Do This Instead |
|-------------|-------------|----------------|
| Print to stdout in library code | Breaks composability | Return data, let caller decide |
| Ignore SIGPIPE | Broken pipes cause panics | Handle or use `reset_sigpipe` |
| `unwrap()` in user-facing paths | Cryptic panic messages | `context("doing X")?` |
| Colors on non-TTY | Breaks piping to files | Check `atty::is(Stream::Stdout)` or use `console` crate |
| One giant `main()` | Untestable | Extract logic into lib.rs |
| Hard-coded paths | Not portable | Use `dirs` crate for config/data dirs |

**Composability checklist**:
1. Data on stdout, UI on stderr
2. Exit codes: 0 success, 1 error, 2 usage
3. Accept stdin when no file arg given (`-` convention)
4. Support `--quiet` for scripts
5. No colors when not a TTY
6. Support `--output -` for stdout

**Skill to practice**: Refactor a CLI to follow all conventions: proper exit codes, anyhow error reporting, --quiet/--verbose flags, shell completions, and stderr for UI output.

---

## Quick Reference

| Skill | Pattern | One-Liner |
|-------|---------|-----------|
| 165 | clap Derive API | `#[derive(Parser)]`, positional/optional/env args, doc-comment help |
| 166 | Subcommands & Enums | `#[derive(Subcommand)]`, nested commands, `flatten` shared groups |
| 167 | Validation & Value Types | `ValueEnum`, custom value parsers, mutually exclusive groups |
| 168 | Layered Config | defaults → config file → env vars → CLI args with config-rs |
| 169 | tracing Structured Logging | `info!(key = value)`, `#[instrument]`, `EnvFilter`, spans |
| 170 | tracing Layers | Registry + multiple layers, JSON output, file appender, OpenTelemetry |
| 171 | Progress & Interaction | indicatif progress bars, dialoguer prompts, stderr for UI |
| 172 | Signal Handling | `ctrl_c()`, CancellationToken, graceful drain, RAII cleanup |
| 173 | Testing CLIs | assert_cmd, trycmd snapshots, env/stdin injection, tempfile configs |
| 174 | Best Practices | Exit codes, anyhow, shell completions, composability, anti-patterns |

---

## Change Log

| Date | Change |
|------|--------|
| 2026-02-20 | Initial version — Skills 165-174 from CLI ecosystem analysis |
