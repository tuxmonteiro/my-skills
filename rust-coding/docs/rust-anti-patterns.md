# Rust Patterns Book — Appendix: Anti-Patterns

> Source: [Rust Patterns Book — Appendix C: Anti-Patterns](https://www.rust-patterns.com/book/33-appendix-c-anti-patterns.html)
>
> This Markdown is a **structured summary/conversion guide**, not a verbatim reproduction of the source page.

## Overview

Rust anti-patterns are recurring approaches that may initially appear convenient but tend to produce bugs, unnecessary complexity, performance problems, or maintenance difficulties.

The source groups them into four categories:

- Common pitfalls
- Performance anti-patterns
- Safety anti-patterns
- API design mistakes

A useful mental model is to treat Rust's ownership, borrowing, and type systems as design tools rather than obstacles.

---

## Common Pitfalls

### 1. Excessive Cloning

**Problem:** Cloning values merely to satisfy the borrow checker.

Typical consequences:

- Extra allocations and copies
- Higher memory consumption
- Increased latency
- Ownership misunderstandings

**Prefer:**

- `&T` for read-only access
- `&mut T` for mutation
- Moving ownership when the original value is no longer needed
- Cloning only when independent ownership is semantically required

```rust
fn process(data: Vec<String>) {
    print_data(&data);
    let transformed = transform_data(&data);
    save_data(data);
}

fn print_data(data: &[String]) {
    for item in data {
        println!("{item}");
    }
}

fn transform_data(data: &[String]) -> Vec<String> {
    data.iter().map(|s| s.to_uppercase()).collect()
}

fn save_data(data: Vec<String>) {
    // Takes ownership.
}
```

### 2. Overusing `Rc` / `Arc`

**Problem:** Using reference counting when ordinary borrowing or ownership would be sufficient.

Potential costs:

- Heap allocation
- Reference-counting overhead
- More complicated ownership semantics
- Reduced clarity of data ownership

**Prefer:** references and straightforward ownership.

Use `Rc`/`Arc` when there is genuine shared ownership. `Arc` is appropriate when that ownership crosses thread boundaries.

```rust
struct Processor<'a> {
    config: &'a Config,
    logger: &'a Logger,
}
```

### 3. Ignoring Iterator Combinators

Manual accumulation can be unnecessarily verbose when the operation naturally maps to iterator combinators.

```rust
fn process_numbers(numbers: &[i32]) -> Vec<i32> {
    numbers
        .iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * 2)
        .collect()
}
```

Use `filter`, `map`, `find`, `sum`, `fold`, and related combinators when they make the intent clearer.

### 4. Deref Coercion Abuse

**Problem:** Implementing `Deref` to simulate inheritance or expose unrelated APIs implicitly.

`Deref` is primarily intended for pointer-like abstractions. Using it as an inheritance mechanism can make APIs surprising and fragile.

**Prefer:** explicit delegation or traits.

```rust
impl Manager {
    fn employee(&self) -> &Employee {
        &self.employee
    }

    fn name(&self) -> &str {
        &self.employee.name
    }
}
```

### 5. `String` vs `&str` Confusion

Avoid requiring owned strings when a borrowed string slice is sufficient.

```rust
fn greet(name: &str) -> String {
    format!("Hello, {name}")
}
```

This accepts both literals and borrowed `String` values without forcing callers to allocate.

Use `String` when ownership is required or when the function creates and returns owned data.

---

## Performance Anti-Patterns

### 6. Collecting Iterators Unnecessarily

Avoid materializing intermediate `Vec`s when the next operation can consume an iterator directly.

```rust
fn process(numbers: &[i32]) -> i32 {
    numbers
        .iter()
        .filter(|&&n| n % 2 == 0)
        .map(|&n| n * 2)
        .sum()
}
```

Use `.collect()` when a concrete collection is actually required, such as when returning it or reusing it multiple times.

### 7. `Vec<T>` When an Array Suffices

For fixed-size data, arrays can avoid heap allocation and communicate size at compile time.

```rust
fn rgb(pixel: u32) -> [u8; 3] {
    [
        ((pixel >> 16) & 0xff) as u8,
        ((pixel >> 8) & 0xff) as u8,
        (pixel & 0xff) as u8,
    ]
}
```

Use `Vec<T>` when the number of elements is genuinely dynamic.

### 8. `HashMap` for Small Fixed Key Sets

A `HashMap` can be excessive for a handful of known values.

```rust
fn status_code(status: &str) -> u16 {
    match status {
        "ok" => 200,
        "not_found" => 404,
        "error" => 500,
        _ => 500,
    }
}
```

Consider arrays, slices, or `match` for small static sets. Use `HashMap` when the data set is sufficiently large or dynamic to justify it.

### 9. Premature String Allocation

Do not convert borrowed strings to owned `String` values before knowing that ownership is necessary.

```rust
fn process_line(line: &str) -> Option<String> {
    if !line.starts_with("ERROR") {
        return None;
    }

    Some(line.to_uppercase())
}
```

For conditional ownership, `Cow<'a, str>` can be useful.

### 10. Boxed Trait Objects Everywhere

`Box<dyn Trait>` provides dynamic dispatch, but it should not be the default when concrete types are known at compile time.

Prefer generics or `impl Trait` when static dispatch is appropriate.

```rust
fn process_twice(data: &str, processor: impl Processor) -> String {
    let once = processor.process(data);
    processor.process(&once)
}
```

Use dynamic dispatch when runtime polymorphism is genuinely required.

---

## Safety Anti-Patterns

### 11. `unsafe` for Convenience

Do not use `unsafe` simply to bypass the borrow checker.

Unsafe code can introduce:

- Undefined behavior
- Aliased mutable references
- Out-of-bounds memory access
- Difficult-to-audit invariants

Prefer safe abstractions such as `split_at_mut`.

```rust
fn get_two_mut(
    data: &mut [String],
    i: usize,
    j: usize,
) -> Option<(&mut String, &mut String)> {
    if i == j || i >= data.len() || j >= data.len() {
        return None;
    }

    if i < j {
        let (left, right) = data.split_at_mut(j);
        Some((&mut left[i], &mut right[0]))
    } else {
        let (left, right) = data.split_at_mut(i);
        Some((&mut right[0], &mut left[j]))
    }
}
```

If `unsafe` is genuinely necessary, document the safety invariants and minimize its scope.

### 12. `unwrap()` / `expect()` in Production Paths

Unchecked unwrapping can turn recoverable failures into panics.

Prefer propagating errors:

```rust
fn load_config(path: &str) -> Result<Config, ConfigError> {
    let contents = std::fs::read_to_string(path)?;
    Config::parse(&contents)
}
```

`unwrap()` and `expect()` can still be appropriate when a condition is truly guaranteed by program invariants, tests, or controlled initialization.

### 13. `RefCell` / `Mutex` Without Consideration

Interior mutability and synchronization primitives are useful, but should not replace normal ownership and borrowing by default.

Potential problems include:

- Runtime borrow checking
- Runtime panics
- Hidden mutation
- Lock contention
- More difficult reasoning about state

Prefer `&mut self` and ordinary ownership when possible.

Use `Mutex`/`RwLock` when shared concurrent mutation is genuinely required.

### 14. Ignoring `Send` / `Sync`

Thread safety is part of Rust's type system.

Do not circumvent `Send`/`Sync` restrictions with `unsafe`.

For shared mutable state across threads, consider:

```rust
use std::sync::{Arc, Mutex};

let data = Arc::new(Mutex::new(Vec::<i32>::new()));
```

Message passing can be preferable when ownership transfer is more natural than shared mutable state.

---

## API Design Mistakes

### 15. Stringly-Typed APIs

Avoid strings when the domain has a finite, meaningful set of values.

Instead of:

```rust
fn set_log_level(level: &str) {
    // Runtime validation.
}
```

prefer:

```rust
#[derive(Debug, Clone, Copy)]
enum LogLevel {
    Debug,
    Info,
    Warn,
    Error,
}

fn set_log_level(level: LogLevel) {
    // Compile-time checked.
}
```

Benefits:

- Compile-time validation
- Better IDE completion
- Better discoverability
- Fewer spelling mistakes
- Explicit domain modeling

### 16. Boolean Parameter Trap

Multiple booleans can make call sites difficult to understand.

Avoid:

```rust
connect("localhost", true, false, true);
```

Prefer enums:

```rust
enum Encryption {
    Encrypted,
    Plaintext,
}

enum ConnectionMode {
    Persistent,
    Transient,
}

enum Verbosity {
    Verbose,
    Quiet,
}
```

Or use a builder when there are many optional configuration choices.

### 17. Leaky Abstractions

Do not expose implementation details that callers should not depend on.

Prefer keeping internal collections private and exposing domain-oriented operations.

```rust
pub struct Database {
    connection_pool: Vec<Connection>,
    cache: HashMap<String, Vec<u8>>,
}

impl Database {
    pub fn query(&self, sql: &str) -> Result<Vec<u8>, Error> {
        // Implementation remains replaceable.
        todo!()
    }
}
```

This preserves the freedom to replace internal data structures without breaking consumers.

### 18. Returning Owned Values When Borrowing Suffices

Avoid cloning fields merely to return them.

Instead of:

```rust
impl User {
    fn name(&self) -> String {
        self.name.clone()
    }
}
```

prefer:

```rust
impl User {
    fn name(&self) -> &str {
        &self.name
    }
}
```

Return owned values when the API creates new data or when ownership is genuinely required.

### 19. Overengineered Generic APIs

Generics should provide real abstraction value.

Avoid unnecessary trait bounds such as:

```rust
fn print_items<I, T>(items: I)
where
    I: IntoIterator<Item = T>,
    T: Display + Debug + Clone + Send + Sync + 'static,
{
    // ...
}
```

If the operation only requires `Display`, keep the constraint small:

```rust
fn print_items<T: std::fmt::Display>(items: &[T]) {
    for item in items {
        println!("{item}");
    }
}
```

Reserve complex bounds for cases where the implementation actually requires them.

---

## Summary: Practical Rules

### Ownership and Borrowing

- Clone only when independent ownership is required.
- Understand the borrow checker before introducing ownership workarounds.
- Prefer references for temporary/read-only access.
- Use `Rc`/`Arc` only for genuine shared ownership.

### Performance

- Avoid intermediate collections in iterator pipelines.
- Use arrays for fixed-size values.
- Avoid unnecessary heap allocation.
- Prefer `&str` until ownership is needed.
- Prefer static dispatch when runtime polymorphism is unnecessary.
- Benchmark before making performance claims.

### Safety

- Treat `unsafe` as a narrowly scoped mechanism, not a workaround.
- Prefer `Result`/`Option` over unchecked panics in recoverable paths.
- Use ordinary borrowing before interior mutability.
- Respect `Send`/`Sync` instead of bypassing them.

### API Design

- Model domain concepts with types.
- Avoid ambiguous boolean arguments.
- Keep implementation details private.
- Return borrowed data when ownership is unnecessary.
- Keep generic bounds as simple as possible.

---

## Recognition Checklist

When reviewing Rust code, ask:

- Is `.clone()` being used because ownership is actually required, or merely to silence the compiler?
- Is `Rc`/`Arc` necessary, or could borrowing solve the problem?
- Could this loop be expressed more clearly with iterator combinators?
- Is `Deref` being used for something other than pointer-like behavior?
- Does this API unnecessarily require `String`?
- Are intermediate `Vec`s being allocated without a need to materialize them?
- Is a fixed-size collection represented as a heap-allocated `Vec`?
- Is a `HashMap` being recreated for a tiny static key set?
- Is a `String` allocated before it is known to be necessary?
- Is dynamic dispatch required?
- Is `unsafe` genuinely necessary and justified?
- Could an error propagate through `Result` instead of panicking?
- Is interior mutability hiding state changes?
- Are `Send` and `Sync` constraints being respected?
- Would an enum communicate intent better than a string or boolean?
- Are implementation details unnecessarily exposed?
- Could a returned `String`/`Vec` be a borrowed value instead?
- Are generic bounds justified by the implementation?

## Source

Original page:

https://www.rust-patterns.com/book/33-appendix-c-anti-patterns.html
