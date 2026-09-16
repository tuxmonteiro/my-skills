# Rust Patterns Book — Builder & API Design

> Source: [Rust Patterns Book — Builder & API Design](https://www.rust-patterns.com/book/05-builder-api-design.html)
>
> This Markdown is a structured conversion of the chapter content.

## Overview

This chapter presents several Rust API-design patterns:

- **Builder Pattern** — flexible and readable construction of complex objects.
- **Typestate Pattern** — compile-time validation of state transitions.
- **Fluent APIs** — ergonomic method chaining.
- **Generic Parameters** — flexible function arguments using traits such as `Into` and `AsRef`.
- **`#[must_use]`** — preventing accidental omission of important return values.

---

# Pattern 1: Builder Pattern Variations

The Builder pattern is useful for constructing complex objects with multiple optional fields or lengthy configuration.

## Problem

Constructors with many parameters can be:

- Confusing
- Error-prone
- Difficult to read at call sites
- Difficult to evolve when new options are added
- Verbose when many parameters are optional

For example, positional arguments do not communicate the meaning of each value particularly well.

## Solution

Use a separate builder object to configure the final object incrementally. A final `.build()` operation constructs the target value.

```rust
Request::builder("https://api.example.com")
    .method("POST")
    .header("Authorization", "Bearer token")
    .body("{"data": "value"}")
    .build();
```

This makes configuration self-documenting.

## Typical Use Cases

- HTTP request construction
- Database connection configuration
- UI component construction
- Application configuration
- Test-data construction

---

## Consuming Builder

The consuming builder is the most common variant.

Each setter:

1. Takes ownership of the builder with `self`.
2. Mutates it.
3. Returns the builder.
4. Allows another method to be chained.

The final `.build()` consumes the builder and produces the target object.

```rust
use std::time::Duration;

#[derive(Debug)]
pub struct Request {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout: Option<Duration>,
    retry_count: u32,
    follow_redirects: bool,
}

pub struct RequestBuilder {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
    timeout: Option<Duration>,
    retry_count: u32,
    follow_redirects: bool,
}

impl Request {
    pub fn builder(url: impl Into<String>) -> RequestBuilder {
        RequestBuilder::new(url)
    }
}

impl RequestBuilder {
    pub fn new(url: impl Into<String>) -> Self {
        Self {
            url: url.into(),
            method: "GET".to_string(),
            headers: Vec::new(),
            body: None,
            timeout: None,
            retry_count: 0,
            follow_redirects: true,
        }
    }

    pub fn method(mut self, method: impl Into<String>) -> Self {
        self.method = method.into();
        self
    }

    pub fn header(
        mut self,
        key: impl Into<String>,
        value: impl Into<String>,
    ) -> Self {
        self.headers.push((key.into(), value.into()));
        self
    }

    pub fn body(mut self, body: impl Into<String>) -> Self {
        self.body = Some(body.into());
        self
    }

    pub fn build(self) -> Request {
        Request {
            url: self.url,
            method: self.method,
            headers: self.headers,
            body: self.body,
            timeout: self.timeout,
            retry_count: self.retry_count,
            follow_redirects: self.follow_redirects,
        }
    }
}
```

Usage:

```rust
let request = Request::builder("https://api.example.com")
    .method("POST")
    .header("Authorization", "Bearer token")
    .body("{"data": "value"}")
    .build();
```

### Characteristics

The consuming builder is especially useful when:

- A builder represents a single construction operation.
- Reuse is not required.
- Ownership transfer naturally matches the lifecycle.
- You want the builder to become unusable after `.build()`.

---

## Builder with Runtime Validation

When some fields are mandatory, the builder can store them as `Option<T>` and validate them in `.build()`.

This moves validation to runtime but keeps the validation centralized.

```rust
#[derive(Debug)]
pub struct Database {
    host: String,
    port: u16,
    username: String,
}

pub struct DatabaseBuilder {
    host: Option<String>,
    port: Option<u16>,
    username: Option<String>,
}

impl DatabaseBuilder {
    pub fn new() -> Self {
        Self {
            host: None,
            port: None,
            username: None,
        }
    }

    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }

    pub fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }

    pub fn username(mut self, username: impl Into<String>) -> Self {
        self.username = Some(username.into());
        self
    }

    pub fn build(self) -> Result<Database, String> {
        let host = self.host.ok_or("host is required")?;
        let port = self.port.ok_or("port is required")?;
        let username = self.username.ok_or("username is required")?;

        Ok(Database {
            host,
            port,
            username,
        })
    }
}
```

Usage:

```rust
let db = DatabaseBuilder::new()
    .host("localhost")
    .port(5432)
    .username("admin")
    .build();

assert!(db.is_ok());

let invalid = DatabaseBuilder::new()
    .host("localhost")
    .build();

assert!(invalid.is_err());
```

### Trade-off

Runtime validation is simpler than typestate, but incorrect configurations are detected only when `.build()` executes.

---

## Non-Consuming Mutable Builder

A builder can instead use `&mut self` and return `&mut Self`.

This is useful when a builder needs to be reused or incrementally modified.

```rust
#[derive(Debug)]
pub struct Email {
    to: Vec<String>,
    subject: String,
    body: String,
}

pub struct EmailBuilder {
    to: Vec<String>,
    subject: String,
    body: String,
}

impl EmailBuilder {
    pub fn new() -> Self {
        Self {
            to: Vec::new(),
            subject: String::new(),
            body: String::new(),
        }
    }

    pub fn to(&mut self, email: impl Into<String>) -> &mut Self {
        self.to.push(email.into());
        self
    }

    pub fn subject(&mut self, subject: impl Into<String>) -> &mut Self {
        self.subject = subject.into();
        self
    }

    pub fn body(&mut self, body: impl Into<String>) -> &mut Self {
        self.body = body.into();
        self
    }

    pub fn build(&self) -> Email {
        Email {
            to: self.to.clone(),
            subject: self.subject.clone(),
            body: self.body.clone(),
        }
    }

    pub fn clear(&mut self) {
        self.to.clear();
        self.subject.clear();
        self.body.clear();
    }
}
```

Usage:

```rust
let mut builder = EmailBuilder::new();

builder
    .to("a@example.com")
    .subject("First")
    .body("Hello");

let email1 = builder.build();

builder.clear();

builder
    .to("b@example.com")
    .subject("Second")
    .body("World");

let email2 = builder.build();
```

### Trade-off

A mutable builder can be reused, but building often requires cloning data because the builder remains usable.

---

# Pattern 2: Typestate Pattern

The Typestate pattern encodes an object's state in its type.

Instead of validating state transitions at runtime, the compiler prevents invalid transitions.

## Problem

Traditional state machines may rely on runtime checks:

```rust
if self.state == State::Connected {
    // ...
}
```

This can lead to:

- Forgotten state checks
- Invalid transitions
- Runtime panics
- Complex conditional logic

## Solution

Represent each state with a distinct type.

A transition consumes the object in its current state and returns an object representing the next state.

```text
Disconnected
      |
    connect()
      |
      v
 Connected
      |
     close()
      |
      v
 Disconnected
```

The compiler then prevents methods that are invalid for a given state.

## Why It Matters

Invalid state transitions become compile-time errors rather than runtime errors.

For example, a disconnected connection simply does not have a `send()` method.

### Typical Use Cases

- Database authentication states
- File lifecycle states
- Network protocol state machines
- Builders requiring fields in a particular sequence
- Resource acquisition/release lifecycles

---

## Typestate Connection Example

```rust
use std::marker::PhantomData;
use std::io::{self, Write};
use std::net::TcpStream;

#[derive(Debug)]
struct Disconnected;

#[derive(Debug)]
struct Connected;

struct Connection<State> {
    stream: Option<TcpStream>,
    _state: PhantomData<State>,
}

impl Connection<Disconnected> {
    fn new() -> Self {
        Self {
            stream: None,
            _state: PhantomData,
        }
    }

    fn connect(
        self,
        addr: &str,
    ) -> io::Result<Connection<Connected>> {
        let stream = TcpStream::connect(addr)?;

        println!("Connected to {addr}");

        Ok(Connection {
            stream: Some(stream),
            _state: PhantomData,
        })
    }
}

impl Connection<Connected> {
    fn send(&mut self, data: &[u8]) -> io::Result<()> {
        let stream = self
            .stream
            .as_mut()
            .expect("Stream must exist in Connected state");

        stream.write_all(data)?;

        println!(
            "Sent data: {}",
            String::from_utf8_lossy(data)
        );

        Ok(())
    }

    fn close(self) -> Connection<Disconnected> {
        if let Some(stream) = self.stream {
            drop(stream);
        }

        println!("Connection closed.");

        Connection {
            stream: None,
            _state: PhantomData,
        }
    }
}
```

Usage:

```rust
let conn = Connection::new();

// conn.send(b"hello");
// ERROR: `send` does not exist for Connection<Disconnected>

let mut connected = conn.connect("127.0.0.1:8080")?;

connected.send(b"hello")?;

let _closed = connected.close();
```

The important property is that `send()` exists only for `Connection<Connected>`.

---

## Typestate Builder

Typestate can also enforce required builder fields at compile time.

Instead of:

```rust
builder.build()
```

returning a runtime error, the compiler can make `.build()` unavailable until all mandatory fields have been provided.

### State Markers

```rust
use std::marker::PhantomData;

#[derive(Default)]
struct NoName;

#[derive(Default)]
struct HasName;

#[derive(Default)]
struct NoEmail;

#[derive(Default)]
struct HasEmail;
```

### Domain Object

```rust
#[derive(Debug)]
struct User {
    name: String,
    email: String,
}
```

### Generic Builder

```rust
struct UserBuilder<NameState, EmailState> {
    name: Option<String>,
    email: Option<String>,
    _name_state: PhantomData<NameState>,
    _email_state: PhantomData<EmailState>,
}
```

### Initial State

```rust
impl Default for UserBuilder<NoName, NoEmail> {
    fn default() -> Self {
        Self {
            name: None,
            email: None,
            _name_state: PhantomData,
            _email_state: PhantomData,
        }
    }
}
```

### State Transition: Name

```rust
impl<E> UserBuilder<NoName, E> {
    fn name(
        self,
        name: impl Into<String>,
    ) -> UserBuilder<HasName, E> {
        UserBuilder {
            name: Some(name.into()),
            email: self.email,
            _name_state: PhantomData,
            _email_state: PhantomData,
        }
    }
}
```

### State Transition: Email

```rust
impl<N> UserBuilder<N, NoEmail> {
    fn email(
        self,
        email: impl Into<String>,
    ) -> UserBuilder<N, HasEmail> {
        UserBuilder {
            name: self.name,
            email: Some(email.into()),
            _name_state: PhantomData,
            _email_state: PhantomData,
        }
    }
}
```

### Final State

Only the fully configured state exposes `.build()`:

```rust
impl UserBuilder<HasName, HasEmail> {
    fn build(self) -> User {
        User {
            name: self.name.expect("guaranteed by typestate"),
            email: self.email.expect("guaranteed by typestate"),
        }
    }
}
```

Usage:

```rust
let user = UserBuilder::default()
    .name("Alice")
    .email("alice@example.com")
    .build();
```

This does not compile:

```rust
// UserBuilder::default()
//     .name("Bob")
//     .build();
// ERROR: `build` is not available until email is set.
```

### Typestate Trade-offs

**Advantages:**

- Invalid states become unrepresentable.
- Validation happens at compile time.
- State transitions are explicit.
- No runtime state-checking overhead is required.

**Costs:**

- More types and generic parameters.
- More complex implementation.
- Potentially more complicated compiler diagnostics.
- May be excessive for simple builders.

Use typestate when compile-time guarantees provide meaningful value.

---

# Pattern 3: `#[must_use]` for Critical Return Values

The `#[must_use]` attribute tells the compiler that a return value should not be silently discarded.

## Problem

Some functions return values that are important to program correctness.

Examples:

- `Result`
- `Option`
- Builders
- Transaction guards
- Resource handles
- Futures

Ignoring these values can result in silent failures.

## Solution

Apply `#[must_use]` to functions or types.

```rust
#[must_use = "this Result may be an error to handle"]
pub fn connect_to_db() -> Result<(), &'static str> {
    Err("Failed to connect")
}
```

A type can also be marked:

```rust
#[must_use = "a builder does nothing unless you call `.build()`"]
pub struct ConnectionBuilder;

impl ConnectionBuilder {
    pub fn new() -> Self {
        ConnectionBuilder
    }

    pub fn build(self) {}
}
```

### Usage

The compiler warns when the return value is ignored:

```rust
// Warning: unused `ConnectionBuilder`
ConnectionBuilder::new();
```

Correct usage:

```rust
ConnectionBuilder::new().build();
```

For `Result`:

```rust
if let Err(error) = connect_to_db() {
    println!("Error: {error}");
}
```

## Why `#[must_use]` Matters

Silent errors can be difficult to detect.

The attribute turns an accidental omission into a compiler warning.

The Rust standard library uses this mechanism for important types such as:

- `Result`
- `Option`
- Iterators

## Typical Use Cases

Use `#[must_use]` for:

- Fallible operations
- Builders
- Resource handles
- Transaction objects
- Futures
- Lazy computations
- Values whose destruction without explicit handling is likely a programming error

---

# Pattern Selection Guide

| Requirement | Recommended Pattern |
|---|---|
| Many optional configuration values | Builder |
| Fluent configuration API | Consuming or mutable Builder |
| Builder needs reuse | Mutable Builder |
| Required fields need runtime validation | Builder + `Result` |
| Required fields should be compile-time enforced | Typestate Builder |
| Object has meaningful lifecycle states | Typestate |
| Important return value must not be ignored | `#[must_use]` |
| Simple configuration with few fields | Direct constructor may be preferable |
| Runtime state is genuinely dynamic | Conventional state machine may be simpler |

---

# Design Considerations

## Prefer the Simplest Pattern That Provides the Required Guarantee

Do not introduce typestate merely because it is technically possible.

For a small configuration object, a normal constructor may be clearer:

```rust
Config::new(host, port)
```

For many optional parameters:

```rust
Config::builder()
    .host(host)
    .port(port)
    .timeout(timeout)
    .build()
```

For strict compile-time requirements:

```rust
UserBuilder::default()
    .name(name)
    .email(email)
    .build()
```

Choose the pattern based on the guarantees and ergonomics the API actually needs.

## Compile-Time vs Runtime Validation

| Approach | Validation | Complexity |
|---|---|---|
| Constructor | Compile time for type correctness | Low |
| Builder + `Result` | Runtime | Low/Medium |
| Typestate | Compile time | High |
| Runtime state machine | Runtime | Medium |

Typestate is particularly useful when invalid states represent programming errors rather than user-provided invalid input.

## Consuming vs Mutable Builders

### Consuming Builder

```rust
builder
    .host("localhost")
    .port(5432)
    .build();
```

Advantages:

- Ownership naturally follows construction.
- No need to clone builder-owned values during `build()`.
- Builder cannot accidentally be reused.
- Strong linear construction semantics.

### Mutable Builder

```rust
let mut builder = Builder::new();

builder.host("localhost");
builder.port(5432);

let first = builder.build();

builder.clear();
```

Advantages:

- Reusable.
- Useful for incremental construction.
- Useful when building multiple similar objects.

Cost:

- `build()` commonly requires cloning values because the builder remains valid.

---

# Practical Review Checklist

When designing or reviewing a Rust API, ask:

- Does the constructor have too many parameters?
- Would named builder methods make the call site clearer?
- Should the builder consume itself?
- Does the builder need to be reusable?
- Are mandatory fields validated at runtime or compile time?
- Would typestate materially improve correctness?
- Is typestate adding unnecessary generic complexity?
- Are state transitions represented by types where appropriate?
- Are important return values marked `#[must_use]`?
- Can callers accidentally ignore an operation that should be handled?
- Does the API use `Into`/`AsRef` where flexible argument ownership is useful?
- Are method chains readable rather than merely clever?
- Is the abstraction simpler than the problem it solves?

---

# Key Takeaways

1. **Use builders for complex construction.**
2. **Use consuming builders when a builder represents one construction lifecycle.**
3. **Use mutable builders when reuse is important.**
4. **Use `Result` for runtime validation when invalid configuration is possible at runtime.**
5. **Use typestate when invalid states should be impossible to represent or use.**
6. **Use `#[must_use]` for values that must not be silently discarded.**
7. **Prefer the simplest API that provides the required correctness guarantees.**
8. **Treat compile-time guarantees as a design tool, not a reason to add unnecessary type complexity.**

## Source

Original chapter:

https://www.rust-patterns.com/book/05-builder-api-design.html
