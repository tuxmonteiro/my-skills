---
name: rust-coding
description: Unified Rust engineering skill covering idiomatic patterns (ownership, error handling, enums, traits, concurrency, unsafe code) and baseline coding standards (naming, module organization, documentation, testing, code smells). Use this whenever writing, reviewing, or refactoring Rust code, designing crate/module structure, setting up lint/format/test conventions, or onboarding contributors to a Rust codebase. Merges and supersedes the separate "rust-patterns" and "coding-standards" skills.
---

# Rust Coding Standards & Patterns

A single, consolidated engineering baseline for Rust: idiomatic language patterns plus the naming,
structure, and review conventions that keep a codebase maintainable. This skill merges two
previously separate skills (`rust-patterns` and `coding-standards`) into one source of truth, so
there is no ambiguity about which one to follow.

## When to Use

- Writing new Rust code
- Reviewing Rust code
- Refactoring existing Rust code
- Designing crate/workspace structure and module layout
- Setting up lint, formatter, documentation, and testing standards
- Onboarding contributors

Not the primary source for: web-framework-specific patterns, embedded development, or
project-specific architecture — treat this as the shared baseline underneath those.

## How It Works

This skill enforces idiomatic Rust conventions across the areas below: ownership and borrowing to
prevent data races at compile time; `Result`/`?` error propagation with `thiserror` for libraries
and `anyhow` for applications; enums and exhaustive pattern matching to make illegal states
unrepresentable; traits and generics for zero-cost abstraction; safe concurrency via
`Arc<Mutex<T>>`, channels, and async/await; minimal `pub` surfaces organized by domain; plus the
naming, documentation, testing, and code-review conventions that keep all of the above consistent
across a team.

---

# Core Engineering Principles

## 1. Readability First

- Code is read more than written.
- Prefer expressive names; let the type system explain intent.
- Follow standard Rust idioms; prefer self-documenting code.

## 2. KISS

- Prefer the simplest ownership model.
- Avoid unnecessary abstractions; favor explicit code over clever tricks.
- Don't optimize before profiling.

## 3. DRY

- Extract reusable functions; prefer traits over duplicated implementations.
- Share common utilities; avoid copy-paste logic.

## 4. YAGNI

- Avoid speculative generic abstractions.
- Don't introduce traits until multiple implementations exist.
- Keep APIs minimal; refactor when requirements evolve.

---

# Naming Conventions

## Variables

```rust
// GOOD
let market_query = "election";
let total_revenue = 1_000;
let is_authenticated = true;

// BAD
let q = "election";
let x = 1000;
let flag = true;
```

## Functions

```rust
// GOOD
fn calculate_similarity(a: &[f32], b: &[f32]) -> f32 {}
fn fetch_market(id: MarketId) -> Result<Market> {}
fn is_valid_email(email: &str) -> bool {}

// BAD
fn similarity() {}
fn market() {}
fn email() {}
```

## Types

```rust
struct Market {}
enum MarketStatus {}
trait Repository {}
type MarketId = Uuid;
```

- Types use `PascalCase`.
- Traits are nouns.
- Modules use `snake_case`.
- Constants use `SCREAMING_SNAKE_CASE`.

---

# Ownership and Borrowing

Rust's ownership system prevents data races and memory bugs at compile time. Immutability is the
default — only use `mut` when necessary, and prefer borrowing over moving or cloning.

```rust
// Good: Pass references when you don't need ownership
fn process(data: &[u8]) -> usize {
    data.len()
}

// Good: Take ownership only when you need to store or consume
fn store(data: Vec<u8>) -> Record {
    Record { payload: data }
}

// Bad: Cloning unnecessarily to avoid borrow checker
fn process_bad(data: &Vec<u8>) -> usize {
    let cloned = data.clone(); // Wasteful — just borrow
    cloned.len()
}
```

```rust
// GOOD: mut only when the value actually changes
let mut count = 0;
count += 1;

// BAD: mut on a binding that's never mutated
let mut config = load_config();
// never mutated
```

Prefer borrowing over owned parameters:

```rust
// GOOD
fn display(name: &str) {}
fn process(user: &User) {}

// BAD — forces an unnecessary allocation/move at every call site
fn display(name: String) {}
```

Move ownership only when required. Only clone when ownership genuinely cannot be shared safely —
never clone just to silence the borrow checker without understanding why:

```rust
// Bad: excessive clone chain
user.clone().profile.clone().email.clone()
```

### Use `Cow` for Flexible Ownership

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains(' ') {
        Cow::Owned(input.replace(' ', "_"))
    } else {
        Cow::Borrowed(input) // Zero-cost when no mutation needed
    }
}
```

---

# Error Handling

Always return structured errors and propagate with `?`. Avoid `panic!`/`unwrap()`/`expect()` in
production and library code.

```rust
// Good: Propagate errors with context
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let content = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read config from {path}"))?;
    let config: Config = toml::from_str(&content)
        .with_context(|| format!("failed to parse config from {path}"))?;
    Ok(config)
}

// Bad: Panics on error
fn load_config_bad(path: &str) -> Config {
    let content = std::fs::read_to_string(path).unwrap(); // Panics!
    toml::from_str(&content).unwrap()
}
```

## Library Errors with `thiserror`, Application Errors with `anyhow`

```rust
// Library code: structured, typed errors
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StorageError {
    #[error("record not found: {id}")]
    NotFound { id: String },
    #[error("connection failed")]
    Connection(#[from] std::io::Error),
    #[error("invalid data: {0}")]
    InvalidData(String),
}

// Application code: flexible error handling
use anyhow::{bail, Result};

fn run() -> Result<()> {
    let config = load_config("app.toml")?;
    if config.workers == 0 {
        bail!("worker count must be > 0");
    }
    Ok(())
}
```

Never silently discard a `Result` — respect `#[must_use]`:

```rust
// Bad: Ignoring must_use warnings
let _ = validate(input); // Silently discarding a Result
```

## `Option` Combinators Over Nested Matching

Prefer combinators when they stay readable; fall back to explicit `match` and avoid nested
`if let` when combinators get hard to follow.

```rust
// Good: Combinator chain
fn find_user_email(users: &[User], id: u64) -> Option<String> {
    users.iter()
        .find(|u| u.id == id)
        .map(|u| u.email.clone())
}

user.email.as_deref().unwrap_or("unknown");

// Bad: Deeply nested matching
fn find_user_email_bad(users: &[User], id: u64) -> Option<String> {
    match users.iter().find(|u| u.id == id) {
        Some(user) => match &user.email {
            email => Some(email.clone()),
        },
        None => None,
    }
}
```

- Use map, and_then, unwrap_or_else.
- Avoid excessive if let Some(x) = y nesting. - Better: let value = y.ok_or(MyError::Missing)?.process();

---

# Enums and Pattern Matching

## Model States as Enums

Make illegal states unrepresentable:

```rust
// Good: Impossible states are unrepresentable
enum ConnectionState {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session_id: String },
    Failed { reason: String, retries: u32 },
}

fn handle(state: &ConnectionState) {
    match state {
        ConnectionState::Disconnected => connect(),
        ConnectionState::Connecting { attempt } if *attempt > 3 => abort(),
        ConnectionState::Connecting { .. } => wait(),
        ConnectionState::Connected { session_id } => use_session(session_id),
        ConnectionState::Failed { retries, .. } if *retries < 5 => retry(),
        ConnectionState::Failed { reason, .. } => log_failure(reason),
    }
}
```

## Exhaustive Matching — No Catch-All for Business Logic

Avoid wildcard matches that hide future enum variants unless the omission is intentional and
documented.

```rust
// Good: Handle every variant explicitly
match command {
    Command::Start => start_service(),
    Command::Stop => stop_service(),
    Command::Restart => restart_service(),
    // Adding a new variant forces handling here
}

// Bad: Wildcard hides new variants
match command {
    Command::Start => start_service(),
    _ => {} // Silently ignores Stop, Restart, and future variants
}
```

---

# Traits and Generics

Keep traits small and focused (Interface Segregation); prefer composition over inheritance-like
mega traits.

```rust
trait MarketRepository {
    fn find(&self, id: MarketId) -> Result<Market>;
}
```

## Accept Generics, Return Concrete Types

```rust
// Good: Generic input, concrete output
fn read_all(reader: &mut impl Read) -> std::io::Result<Vec<u8>> {
    let mut buf = Vec::new();
    reader.read_to_end(&mut buf)?;
    Ok(buf)
}

// Good: Trait bounds for multiple constraints
fn process<T: Display + Send + 'static>(item: T) -> String {
    format!("processed: {item}")
}
```

Do not introduce generic parameters until they provide clear, demonstrated value (YAGNI applies to
generics too).

## Trait Objects for Dynamic Dispatch

```rust
// Use when you need heterogeneous collections or plugin systems
trait Handler: Send + Sync {
    fn handle(&self, request: &Request) -> Response;
}

struct Router {
    handlers: Vec<Box<dyn Handler>>,
}

// Use generics when you need performance (monomorphization)
fn fast_process<H: Handler>(handler: &H, request: &Request) -> Response {
    handler.handle(request)
}
```

Note: for library-facing errors, prefer `thiserror`-defined enums over `Box<dyn std::error::Error>`
(see Anti-Patterns).

## Newtype Pattern for Type Safety

```rust
// Good: Distinct types prevent mixing up arguments
struct UserId(u64);
struct OrderId(u64);

fn get_order(user: UserId, order: OrderId) -> Result<Order> {
    // Can't accidentally swap user and order IDs
    todo!()
}

// Bad: Easy to swap arguments
fn get_order_bad(user_id: u64, order_id: u64) -> Result<Order> {
    todo!()
}
```

---

# Structs and Data Modeling

## Builder Pattern for Complex Construction

```rust
struct ServerConfig {
    host: String,
    port: u16,
    max_connections: usize,
}

impl ServerConfig {
    fn builder(host: impl Into<String>, port: u16) -> ServerConfigBuilder {
        ServerConfigBuilder { host: host.into(), port, max_connections: 100 }
    }
}

struct ServerConfigBuilder { host: String, port: u16, max_connections: usize }

impl ServerConfigBuilder {
    fn max_connections(mut self, n: usize) -> Self { self.max_connections = n; self }
    fn build(self) -> ServerConfig {
        ServerConfig { host: self.host, port: self.port, max_connections: self.max_connections }
    }
}

// Usage: ServerConfig::builder("localhost", 8080).max_connections(200).build()
```

---

# Iterators and Closures

Prefer iterator chains over manual loops when they improve readability — but don't sacrifice
clarity for functional style just to be "idiomatic."

```rust
// Good: Declarative, lazy, composable
let active_emails: Vec<String> = users.iter()
    .filter(|u| u.is_active)
    .map(|u| u.email.clone())
    .collect();

// Bad: Imperative accumulation
let mut active_emails = Vec::new();
for user in &users {
    if user.is_active {
        active_emails.push(user.email.clone());
    }
}
```

## Use `collect()` with Type Annotation

```rust
// Collect into different types
let names: Vec<_> = items.iter().map(|i| &i.name).collect();
let lookup: HashMap<_, _> = items.iter().map(|i| (i.id, i)).collect();
let combined: String = parts.iter().copied().collect();

// Collect Results — short-circuits on first error
let parsed: Result<Vec<i32>, _> = strings.iter().map(|s| s.parse()).collect();
```

---

# Concurrency

## `Arc<Mutex<T>>` for Shared Mutable State

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));
let handles: Vec<_> = (0..10).map(|_| {
    let counter = Arc::clone(&counter);
    std::thread::spawn(move || {
        let mut num = counter.lock().expect("mutex poisoned");
        *num += 1;
    })
}).collect();

for handle in handles {
    handle.join().expect("worker thread panicked");
}
```

## Channels for Message Passing

```rust
use std::sync::mpsc;

let (tx, rx) = mpsc::sync_channel(16); // Bounded channel with backpressure

for i in 0..5 {
    let tx = tx.clone();
    std::thread::spawn(move || {
        tx.send(format!("message {i}")).expect("receiver disconnected");
    });
}
drop(tx); // Close sender so rx iterator terminates

for msg in rx {
    println!("{msg}");
}
```

## Async: Structured Concurrency with Tokio

Prefer structured concurrency — run independent work concurrently instead of awaiting sequentially:

```rust
// Good: independent tasks run concurrently
let (users, markets) =
    tokio::join!(load_users(), load_markets());

// Bad: awaiting sequentially when the two calls don't depend on each other
let users = load_users().await;
let markets = load_markets().await;
```

```rust
use tokio::time::Duration;

async fn fetch_with_timeout(url: &str) -> Result<String> {
    let response = tokio::time::timeout(
        Duration::from_secs(5),
        reqwest::get(url),
    )
    .await
    .context("request timed out")?
    .context("request failed")?;

    response.text().await.context("failed to read body")
}

// Spawn concurrent tasks
async fn fetch_all(urls: Vec<String>) -> Vec<Result<String>> {
    let handles: Vec<_> = urls.into_iter()
        .map(|url| tokio::spawn(async move {
            fetch_with_timeout(&url).await
        }))
        .collect();

    let mut results = Vec::with_capacity(handles.len());
    for handle in handles {
        results.push(handle.await.unwrap_or_else(|e| panic!("spawned task panicked: {e}")));
    }
    results
}
```

Never block the executor with synchronous sleeps or blocking I/O inside an `async fn` (see
Anti-Patterns).

---

# Unsafe Code

## When Unsafe Is Acceptable

```rust
// Acceptable: FFI boundary with documented invariants (Rust 2024+)
/// # Safety
/// `ptr` must be a valid, aligned pointer to an initialized `Widget`.
unsafe fn widget_from_raw<'a>(ptr: *const Widget) -> &'a Widget {
    // SAFETY: caller guarantees ptr is valid and aligned
    unsafe { &*ptr }
}

// Acceptable: Performance-critical path with proof of correctness
// SAFETY: index is always < len due to the loop bound
unsafe { slice.get_unchecked(index) }
```

## When Unsafe Is NOT Acceptable

- Using `unsafe` to bypass the borrow checker.
- Using `unsafe` purely for convenience.
- Using `unsafe` without a `// SAFETY:` comment documenting why it's sound.
- Transmuting between unrelated types.

---

# Module System and Crate Structure

> **Note on reconciling structure:** the two source skills recommended different top-level layouts
> — one organized purely **by domain** (`auth/`, `orders/`, `db/`), the other organized **by
> architectural layer** (`domain/`, `application/`, `infrastructure/`, `api/`). These aren't
> mutually exclusive; the reconciled guidance below is domain-first at the top level, with
> layering used only *inside* a domain module if that domain is large enough to need it. Default
> to the simpler domain-first layout (YAGNI) and only add layered sub-structure when a domain
> module actually grows unwieldy.

## Preferred: Organize by Domain, Not by Type

```text
my_app/
├── src/
│   ├── main.rs
│   ├── lib.rs
│   ├── auth/          # Domain module
│   │   ├── mod.rs
│   │   ├── token.rs
│   │   └── middleware.rs
│   ├── orders/        # Domain module
│   │   ├── mod.rs
│   │   ├── model.rs
│   │   └── service.rs
│   └── db/            # Infrastructure
│       ├── mod.rs
│       └── pool.rs
├── tests/             # Integration tests
├── benches/           # Benchmarks
└── Cargo.toml
```

## When a Domain Grows: Layer Inside It, Not Across the Whole Crate

If a single domain becomes large enough that flat files stop being readable, introduce layers
*within* that domain module rather than restructuring the whole crate by layer:

```text
src/
├── orders/
│   ├── mod.rs
│   ├── domain.rs        # entities, value objects
│   ├── application.rs   # use-cases / services
│   ├── infrastructure.rs# persistence, external calls
│   └── api.rs           # handlers/controllers for this domain
├── auth/
├── db/
├── errors.rs
├── config.rs
├── lib.rs
└── main.rs
```

Avoid a crate-wide `domain/ application/ infrastructure/ api/ services/ models/` split by default —
it scatters a single feature across five directories and makes YAGNI-driven refactors harder. Reach
for it only in large, multi-team services where the layering itself is a deliberate architectural
boundary (e.g. hexagonal/clean architecture with enforced dependency direction).

Keep modules cohesive; prefer multiple small modules over massive files.

## Visibility — Expose Minimally

```rust
// Good: pub(crate) for internal sharing
pub(crate) fn validate_input(input: &str) -> bool {
    !input.is_empty()
}

// Good: Re-export public API from lib.rs
pub mod auth;
pub use auth::AuthMiddleware;

// Bad: Making everything pub
pub fn internal_helper() {} // Should be pub(crate) or private
```

---

# When using Pest - general purpose parser written in Rust

### Syntax Constraints

1.  **Syntactic vs Lexical**:
    - Atomic rules (`rule @{ ... }`) generally do NOT consume internal whitespace.
    - Compound rules (`rule = { ... }`) DO consume whitespace implicitly if `WHITESPACE` is defined.
2.  **Special Rules**:
    - `WHITESPACE = _{ " " | "\t" | "\n" }` (Underscore `_` makes it silent).
    - `COMMENT = _{ "//" ~ (!NEWLINE ~ ANY)* }`
3.  **Anchors**:
    - Always start the top-level rule with `SOI` (Start of Input) and end with `EOI`.
    - _Example_: `file = { SOI ~ (stmt)* ~ EOI }`
4.  **Greediness**: - `*` and `+` are eager. - Ordered choice `|` is first-match-wins. Put specific matches first (e.g., `"<=" | "<"`).

# Documentation

Public APIs should use rustdoc. Explain **why**, not **what**; avoid redundant comments.

```rust
/// Returns the market by identifier.
///
/// # Errors
///
/// Returns an error if the market does not exist.
pub fn find(id: MarketId) -> Result<Market> {}
```

---

# Testing

**Skill "rust-tdd" is the priority source for guidance on creating and reviewing Rust tests.**

Use the AAA (Arrange/Act/Assert) pattern, and name tests to describe behavior, not implementation.

```rust
#[test]
fn returns_empty_when_repository_is_empty() {
    // Arrange

    // Act

    // Assert
}
```

```
// GOOD test names
returns_market_when_exists
returns_error_when_market_missing
rejects_invalid_email

// BAD test names
test
works
```

Relevant `cargo` commands:

```bash
cargo test
cargo test -- --nocapture     # Show println output
cargo test --lib              # Unit tests only
cargo test --test integration # Integration tests only
```

---

# Debug

if "Runtime panic", "Logic error", or "Wrong output", follow these steps:

1.  **Reproduction**:
    - Can you write a test case that fails?
    - If not, create a minimal reproducible example (MRE).
2.  **Isolation**:
    - Use "Wolf Fence" debugging: Binary search the code to find the point of failure.
    - Insert `dbg!()` macros (better than `println!`).
3.  **Resolution**: - Once isolated, fix the logic. - Remove all `dbg!()` calls before final commit.

# Tooling Integration

```bash
# Build and check
cargo build
cargo check              # Fast type checking without codegen
cargo clippy --all-targets --all-features   # Lints and suggestions
cargo fmt                # Format code — never manually fight rustfmt

# Dependencies
cargo audit               # Security audit
cargo tree                # Dependency tree
cargo update               # Update dependencies

# Performance
cargo bench               # Run benchmarks
```

- Treat Clippy warnings as errors by default; avoid suppressing a lint unless there's a documented
  justification in a comment next to the `#[allow(...)]`.
- Never manually fight `rustfmt` — configure it in `rustfmt.toml` if defaults don't fit, don't
  hand-format around it.

---

# Performance

- Prefer borrowing over owned/cloned data.
- Reserve capacity when the size is known: `Vec::with_capacity(100)`.
- Avoid unnecessary allocations; prefer slices over owned collections.
- Profile before optimizing — don't guess at hot paths.

---

# Dependency Guidelines

- Prefer the standard library over pulling in a crate for something std already covers well.
- Minimize external dependencies.
- Choose mature, actively maintained libraries.
- Audit transitive dependencies periodically (`cargo audit`, `cargo tree`).

---

# Code Smells

## Long Functions

Keep functions generally below ~50 lines; extract responsibilities into helper functions.

## Deep Nesting

Prefer early returns:

```rust
if !user.is_admin() {
    return Err(Error::Unauthorized);
}

if !market.active {
    return Err(Error::Inactive);
}
```

## Excessive Clone

```rust
// Bad
user.clone().profile.clone().email.clone()
```

Prefer borrowing instead.

## Large Traits

Split traits following Interface Segregation.

## Overusing Generics

Do not introduce generic parameters until they provide clear value.

## Magic Numbers

```rust
const MAX_RETRIES: u8 = 3;
const CACHE_TTL: Duration = Duration::from_secs(60);
```

Avoid unexplained literals.

---

# Anti-Patterns to Avoid

```rust
// Bad: .unwrap() in production code
let value = map.get("key").unwrap();

// Bad: .clone() to satisfy borrow checker without understanding why
let data = expensive_data.clone();
process(&original, &data);

// Bad: Using String when &str suffices
fn greet(name: String) { /* should be &str */ }

// Bad: Box<dyn Error> in libraries (use thiserror instead)
fn parse(input: &str) -> Result<Data, Box<dyn std::error::Error>> { todo!() }

// Bad: Ignoring must_use warnings
let _ = validate(input); // Silently discarding a Result

// Bad: Blocking in async context
async fn bad_async() {
    std::thread::sleep(Duration::from_secs(1)); // Blocks the executor!
    // Use: tokio::time::sleep(Duration::from_secs(1)).await;
}
```

---

# Quick Reference: Rust Idioms

| Idiom | Description |
|-------|-------------|
| Borrow, don't clone | Pass `&T` instead of cloning unless ownership is needed |
| Make illegal states unrepresentable | Use enums to model valid states only |
| `?` over `unwrap()` | Propagate errors, never panic in library/production code |
| Parse, don't validate | Convert unstructured data to typed structs at the boundary |
| Newtype for type safety | Wrap primitives in newtypes to prevent argument swaps |
| Prefer iterators over loops | Declarative chains are clearer and often faster — but not at the cost of clarity |
| `#[must_use]` on Results | Ensure callers handle return values |
| `Cow` for flexible ownership | Avoid allocations when borrowing suffices |
| Exhaustive matching | No wildcard `_` for business-critical enums |
| Minimal `pub` surface | Use `pub(crate)` for internal APIs |
| Domain-first modules | Organize by domain by default; layer *within* a domain only if it grows large |
| Structured concurrency | Prefer `tokio::join!`/`spawn` over sequential `.await` for independent work |

---

# Code Review Checklist

- Idiomatic ownership? Minimal cloning?
- Clear naming (no `x`, `q`, `flag`)?
- Small functions (~<50 lines), shallow nesting (early returns)?
- Proper `Result` usage — no `unwrap()`/`expect()` in library code?
- Exhaustive matches on business-critical enums (no silent wildcard)?
- `thiserror` for library errors, `anyhow` for application entrypoints?
- Structured concurrency used for independent async work?
- No blocking calls inside `async fn`?
- `unsafe` blocks each carry a `// SAFETY:` comment?
- `rustfmt` clean? `cargo clippy --all-targets --all-features` clean?
- Tests included, named for behavior, following AAA?
- Public APIs documented with rustdoc, explaining *why*?
- No premature abstractions or generics (YAGNI)?
- No unnecessary allocations; capacity reserved where size is known?
- Module structure is domain-first, not scattered by architectural layer, unless the crate has a
  deliberate layered-architecture boundary?

---

# Golden Rules

1. Trust the ownership model.
2. Prefer explicitness over cleverness.
3. Keep traits small.
4. Return `Result`, don't panic.
5. Let the compiler enforce correctness.
6. Optimize after measuring.
7. Keep modules cohesive — organize by domain first.
8. Follow Clippy unless there's a documented reason not to.
9. Write code that another developer immediately recognizes as idiomatic.

**Remember**: If it compiles, it's probably correct — but only if you avoid `unwrap()`, minimize
`unsafe`, and let the type system work for you.
