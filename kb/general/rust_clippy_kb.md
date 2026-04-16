# Rust Clippy Warning Patterns

**Created:** 2026-03-21
**Source:** 28 clippy warnings fixed across the Nverse project

This article documents recurring clippy warning categories and their idiomatic fixes, with before/after code examples.

---

## Derivable Default Implementations (`derivable_impls`)

When every field of a struct has a `Default` implementation and the manual `impl Default` just calls `FieldType::default()` or equivalent, clippy flags it as `derivable_impls`.

**Fix:** Replace the manual `impl Default` with `#[derive(Default)]`.

```rust
// Before
struct ConfigFile {
    theme: String,
    timeout: u64,
}

impl Default for ConfigFile {
    fn default() -> Self {
        Self {
            theme: String::default(),
            timeout: 0,
        }
    }
}

// After
#[derive(Default)]
struct ConfigFile {
    theme: String,
    timeout: u64,
}
```

For enums, mark the default variant with `#[default]`:

```rust
// Before
#[derive(Default)]
enum ProcessStatus {
    Unknown,
    Running,
    Stopped,
}

impl Default for ProcessStatus {
    fn default() -> Self { ProcessStatus::Unknown }
}

// After
#[derive(Default)]
enum ProcessStatus {
    #[default]
    Unknown,
    Running,
    Stopped,
}
```

---

## New Without Default (`new_without_default`)

When a type has a `new()` method with no arguments that returns `Self`, clippy expects a corresponding `impl Default` that delegates to it.

**Fix:** Add a `Default` impl that calls `Self::new()`.

```rust
// Before
struct LinuxProcessReader { /* ... */ }

impl LinuxProcessReader {
    pub fn new() -> Self { /* ... */ }
}

// After
impl LinuxProcessReader {
    pub fn new() -> Self { /* ... */ }
}

impl Default for LinuxProcessReader {
    fn default() -> Self {
        Self::new()
    }
}
```

---

## Unnecessary filter_map (`unnecessary_filter_map`)

When every branch of a `filter_map` closure returns `Some(transform(x))` — never `None` — clippy flags it as `unnecessary_filter_map`.

**Fix:** Replace `.filter_map(|x| Some(transform(x)))` with `.map(transform)`.

```rust
// Before
let names: Vec<String> = items
    .filter_map(|x| Some(x.name.clone()))
    .collect();

// After
let names: Vec<String> = items
    .map(|x| x.name.clone())
    .collect();
```

---

## Unnecessary map_or (`unnecessary_map_or`)

Clippy offers two replacements depending on the default value:

| Pattern | Replacement | Min Rust version |
|---------|-------------|-----------------|
| `.map_or(false, \|x\| pred(x))` | `.is_some_and(\|x\| pred(x))` | 1.82 |
| `.map_or(true, \|x\| pred(x))` | `.is_none_or(\|x\| pred(x))` | 1.82 |

**Note:** `is_some_and` and `is_none_or` require **Rust 1.82 or later**. Verify your MSRV before applying.

```rust
// Before
let active = status.map_or(false, |s| s.is_active());
let absent = status.map_or(true,  |s| s.is_empty());

// After
let active = status.is_some_and(|s| s.is_active());
let absent = status.is_none_or(|s| s.is_empty());
```

---

## Trim Then Split Whitespace (`trim_split_whitespace`)

`.trim().split(' ')` does not handle consecutive spaces or other Unicode whitespace. Clippy prefers the combined iterator.

**Fix:** Replace `.trim().split(' ')` with `.split_whitespace()`.

```rust
// Before
let parts: Vec<&str> = line.trim().split(' ').collect();

// After
let parts: Vec<&str> = line.split_whitespace().collect();
```

---

## Manual Split Once (`manual_split_once`)

Using `splitn(2, sep)` followed by two `.next()` calls is the manual equivalent of `split_once`.

**Fix:** Replace the `splitn` + two-`next` pattern with `split_once`.

```rust
// Before
let mut iter = line.splitn(2, ':');
let key = iter.next()?;
let val = iter.next()?;

// After
let (key, val) = line.split_once(':')?;
```

`split_once` is available since Rust 1.52.

---

## Manual Range Contains (`manual_range_contains`)

Chained comparisons like `x >= a && x <= b` are less readable than a range check.

**Fix:** Replace with `(a..=b).contains(&x)`.

```rust
// Before
if x >= 0.0 && x <= 1.0 { /* ... */ }

// After
if (0.0..=1.0).contains(&x) { /* ... */ }
```

---

## Manual Strip (`manual_strip`)

Conditional slicing after `starts_with` / `ends_with` duplicates what `strip_prefix` / `strip_suffix` already do.

**Fix:** Replace with `strip_prefix` or `strip_suffix`.

```rust
// Before
let inner = if s.starts_with('<') { &s[1..] } else { s };

// After
let inner = s.strip_prefix('<').unwrap_or(s);
```

---

## Redundant Closure (`redundant_closure`)

When a closure does nothing but forward its argument to a function or constructor that already has the right signature, the closure is redundant.

**Fix:** Pass the function/constructor directly.

```rust
// Before
results.map_err(|e| InterrogateError::Io(e))

// After
results.map_err(InterrogateError::Io)
```

---

## Redundant Locals (`redundant_locals`)

Rebinding a variable to itself (`let x = x;`) has no effect.

**Fix:** Remove the rebinding.

```rust
// Before
fn run(timeout: Duration) {
    let timeout = timeout;  // redundant
    do_work(timeout);
}

// After
fn run(timeout: Duration) {
    do_work(timeout);
}
```

---

## Type Complexity (`type_complexity`)

When a type alias would make a complex generic or function-pointer type more readable, clippy emits `type_complexity`.

**Fix:** Create a named type alias.

```rust
// Before — in a struct field or function signature
pub refresh_callback: Option<Arc<dyn Fn(&mut HashMap<u32, ProcessInfo>, bool) + Send + Sync>>,

// After
type RefreshCallback = Arc<dyn Fn(&mut HashMap<u32, ProcessInfo>, bool) + Send + Sync>;

pub refresh_callback: Option<RefreshCallback>,
```

---

## Explicit Auto Deref (`explicit_auto_deref`)

Rust auto-derefs in many positions, so manually writing `&*value` is unnecessary when `&value` or just `value` will do.

**Fix:** Remove the explicit `*` deref.

```rust
// Before
some_fn(&*config_store);

// After
some_fn(&config_store);
```

---

## Implicit Saturating Sub (`implicit_saturating_sub`)

A guarded decrement `if x > 0 { x -= 1 }` is a manual saturating subtraction.

**Fix:** Use `saturating_sub`.

```rust
// Before
if frame_counter > 0 {
    frame_counter -= 1;
}

// After
frame_counter = frame_counter.saturating_sub(1);
```

---

## from_str Method Naming Conflict

Defining a method called `from_str` on a type creates an ambiguity with the `std::str::FromStr` trait. Clippy (lint `from_str`) flags this.

Two valid resolutions:

1. **Rename the method** if it has different semantics:
   ```rust
   // Before
   impl ConnectionState {
       pub fn from_str(s: &str) -> Option<Self> { /* ... */ }
   }

   // After
   impl ConnectionState {
       pub fn parse_str(s: &str) -> Option<Self> { /* ... */ }
   }
   ```

2. **Implement the trait** if the semantics match:
   ```rust
   use std::str::FromStr;

   impl FromStr for ConnectionState {
       type Err = ();
       fn from_str(s: &str) -> Result<Self, Self::Err> {
           match s {
               "active" => Ok(Self::Active),
               _ => Err(()),
           }
       }
   }
   ```

Prefer option 2 when the conversion is infallible-enough and callers benefit from `.parse::<ConnectionState>()` syntax.

---

## Running Clippy

```bash
source "$HOME/.cargo/env"
cargo clippy          # all warnings
cargo clippy -- -D warnings   # treat warnings as errors (CI-ready)
```

To check a single module only:
```bash
cargo clippy --lib -- -D warnings 2>&1 | grep "src/state"
```
