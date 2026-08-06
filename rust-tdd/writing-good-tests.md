# Writing Good Tests

**Load this reference when:** writing unit/integration tests, implementing mock objects or double traits, or adding test helpers and fixtures.

## Overview

A test exists to catch a specific break. Two principles govern everything here:

```
1. Every test names the break it catches
2. Every test exercises the real thing

```

Strict TDD produces both naturally: a test written first and watched failing against real code has already proven it can fail, and only earns a mock/trait double when the real dependency proves slow, non-deterministic, or involves external IO.

---

## Principle 1: Name the Break

Before writing the test body, answer: **what production change should make this test fail — and is that change a bug or a decision?** A test earns its place by catching a wrong branch, missing side effect, wrong argument, boundary case, or broken contract.

### Derive Expectations Independently

Use literal values, hand-checked fixtures, or pattern matching (`matches!`). Table-driven tests (or parameterized tests using `rstest`) with literal expected outcomes are the preferred shape. An expectation computed by the code under test — or its helpers — passes no matter what that code does.

```rust
// ❌ Mirror assertion: the same builder computes both sides — always true
let expected = build_search_query(&Filter { tag: "urgent".into() });
assert_eq!(build_search_query(&Filter { tag: "urgent".into() }), expected);

// ✅ Hand-derived literal
assert_eq!(
    build_search_query(&Filter { tag: "urgent".into() }),
    r#"tag:"urgent""#
);

```

### No Change Detectors

If only intentional decisions can fail a test — a constant's value, exact error string formatting (when not part of a public API contract), or private field structures — it fires on redesign and sleeps through bugs. Test the behavior that depends on the decision: not `assert_eq!(MAX_RETRIES, 5)` but "a failing call is retried 5 times, returning `Err(RetryExhausted)` on the 6th attempt."

### Behavior, Not Text

Asserting that a macro, build script (`build.rs`), or configuration file contains an exact line proves only that the source is the source. Run binaries against controlled inputs and assert outputs, side effects, or `ExitCode`/`Command` results.

### Your Code, Not the Standard Library or Crates

Test the contract your code makes at its boundaries — the serialization format of your struct, the SQL query emitted, the response payload produced. Upstream mechanics are their maintainers' tests to write (e.g., asserting that `serde_json::to_string` serializes a standard `String` field — that is `serde`'s test, not yours). When upstream behavior genuinely surprises you, write one narrow characterization test naming the assumption.

Inside your own code: constructors (`new()`), getters, default implementations (`Default`), and trivial delegation earn tests only when they validate, normalize, default, derive, enforce, or cause side effects — otherwise assert the first consumer-visible result that depends on them.

---

### Gate Function

```
BEFORE writing the test body:
  Name the production change that would make this test fail.

  Cannot name one           → Redesign around an observable behavior
  "The source text changed" → Run the artifact and assert its effects
  Only intentional decisions → Change detector; test the behavior
                              that depends on the decision

  Confirm the expected value is derived without the code under test.
  IF it reuses the code's logic or helpers:
    Replace it with a literal, pattern match, or hand-checked fixture

```

---

## Principle 2: Exercise the Real Thing

### The Mock Earns No Assertions

A mock assertion passes when the mock is present and fails when it is absent — it says nothing about the component. Assert the real component's behavior or domain side-effects.

```rust
// ✅ Real behavior (asserting real logic on a returned Result or domain state)
assert!(matches!(service.process_order(&order), Ok(OrderStatus::Processed)));

// ❌ Mock existence (asserting mock object mechanical calls directly when no domain output changed)
assert_eq!(mock_notifier.send_email_call_count(), 1);

```

> **Human Partner Correction:** *"Are we testing the behavior of a mock?"*

### Mock at the Right Level

Learn every side effect of the real method before replacing it with a trait mock or fake implementation. Mock the slow or external operation (e.g., IO, network calls, filesystem access) and keep domain logic real. When unsure, run the test against the real implementation first (e.g., using `tempfile` or an in-memory database like `sqlite::memory`) and observe what actually needs to happen.

```rust
// ❌ The mock swallows file writing that subsequent checks rely on
let mut mock_catalog = MockCatalog::new();
mock_catalog.expect_sync_config().returning(|| Ok(()));

// ✅ Use real temporary storage; only mock the remote HTTP adapter behind a trait
let temp_dir = tempfile::tempdir().unwrap();
let service = ConfigService::new(temp_dir.path(), MockHttpAdapter::new());

```

### Make Doubles Specific

When arguments, call counts, or ordering are part of the contract, assert them strictly using framework expectations (e.g., `mockall` matchers). A fake that accepts anything (`always()`) verifies nothing. Give each branch (`Ok`, `Err(ErrorKind::NotFound)`, malformed) its own explicit response setup.

### Mirror Real Data Completely

Construct mock payloads with complete structures as they exist in production — don't leave critical fields set to `Default::default()` unless that is explicitly what you are testing. Partial or dummy mocks fail silently when downstream code starts reading an omitted field.

### Production Structs Carry Production Methods Only

Cleanup or setup methods that only tests need live in test utilities or helper traits implemented inside `#[cfg(test)]` modules, never as public methods on production structs.

```rust
// ❌ Production code carrying test-only teardown logic
impl DatabasePool {
    pub fn reset_tables_for_testing(&self) { ... } // Do NOT do this
}

// ✅ Test helper module / extension trait inside tests
#[cfg(test)]
mod tests {
    pub trait TestCleanup {
        fn reset_tables(&self);
    }
    impl TestCleanup for DatabasePool {
        fn reset_tables(&self) { ... }
    }
}

```

### Prefer Real Components over Complex Mocks

When mock setup with `mockall` or manual trait doubles outgrows the actual test logic, switch to an integration test under the `tests/` directory using real components (in-memory engines, containerized DBs via `testcontainers`, temporary directories).

> **Human Partner Question:** *"Do we need to be using a mock here?"*

---

### Gate Function

```
BEFORE adding a mock or test helper:
  List the real method's side effects; keep the ones the test
  depends on real — mock at the lowest slow/external trait boundary.

  Mock responses mirror complete, realistic production structs/enums.

  A method only tests call lives inside `#[cfg(test)]` or `tests/common/`, not production API.

  About to assert on the mock itself?
    Unmock it, test the output/state change, or delete the assertion.

```

---

## Tests Ship With the Implementation

The TDD cycle — failing test, minimal implementation, refactor — is what "complete" means. Ship the tests the behavior needs and only those: trivial code and comments earn none, and a test written purely to satisfy code coverage metrics costs maintenance forever.

---

## The Mutation Check

Before finishing, mentally mutate the production code; at least one test should fail for each realistic mutation:

* Inverted boolean or changed match arm
* Wrong constant, numeric offset, or boundary condition (`>` vs `>=`)
* Missing `?` operator or swallowed `Result::Err` / `Option::None`
* Missing state mutation, file write, or event emission
* Default return value (`Default::default()`) swapped in place of real logic
* Missing validation for zero, empty strings, invalid UTF-8, or unauthorized requests

A mutation nothing catches marks the behavior as unprotected — or the test as tautological. *(Tip: Use tools like `cargo-mutants` to automate this verification).*

---

## Quick Reference

| When you... | Do |
| --- | --- |
| Write any test | Name the break it catches — a bug, not a decision |
| Build an expected value | Derive it by hand / literals; never using the code under test |
| Test a CLI tool or macro | Run it against controlled inputs using `assert_cmd` / `trybuild` |
| Reach for a dependency test | Test your boundary contract, not standard library / crate internals |
| Want to assert on a mock | Test the real component's state/output, or unmock it |
| Are about to mock a trait | Learn its side effects; mock the slow/external level |
| Build a mock payload | Mirror the real data struct completely (avoid blank `Default` padding) |
| Need helper/cleanup code | Place it in `#[cfg(test)]` modules or `tests/common/mod.rs` |
| Watch mock setup balloon | Move to integration tests (`tests/`) using real types & `tempfile` |
| Finish a test file | Run the mental mutation check (or `cargo mutants`) |

---

## Warning Signs

* Setup and assertion share the same builder/helper, guaranteeing equality.
* The test can fail only via panic/unwrapper (`.unwrap()`) rather than explicit assertion failures.
* The test fails on every internal refactor, but sleeps through breaking logic changes.
* Expected values are hidden behind nested loops, generators, or complex calculation helpers.
* The test checks internal string representations using regex instead of domain types.
* A method on a production `struct` or `enum` is annotated with `#[doc(hidden)]` because only tests call it.
* Mock setup code dominates more than half the test body.
* Using `mockall` or trait doubles "just to be safe" on fast, pure in-memory code.
