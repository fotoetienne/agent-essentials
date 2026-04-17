# TigerStyle in Rust

How TigerStyle translates to Rust. Most principles map cleanly — Rust's type system and ownership already enforce some of what TigerStyle works hard to assert.

## Safety

### Assertions

- **`debug_assert!`** for invariants asserted in dev/CI, **`assert!`** for critical invariants that must hold in production. Assertion failures panic — aligned with TigerStyle's "crash on programmer error".
- **Split compound asserts:**
  ```rust
  assert!(a);
  assert!(b);
  // not: assert!(a && b);
  ```
- **Implication:** `if a { assert!(b); }`.
- **Compile-time asserts:** `const _: () = assert!(SIZE <= 4096);` or the `static_assertions` crate (but per TigerStyle's "zero dependencies", prefer `const _`).
- **Pair assertions:** encode round-trip invariants — assert before `serialize`, re-assert after `deserialize`.
- **Two per function average** — lean on Rust's pre/post invariants in types, but still encode runtime pre/post for non-trivial logic.

### Control flow

- **Bounded loops.** Prefer `for i in 0..N` and iterator adapters with `.take(N)`. Flag unbounded `while let Some(x) = ch.recv()` in review.
- **No recursion** where iteration works. Rust's lack of TCO in stable makes this extra important.
- **Handle every `Result`/`Option`.** `#[must_use]` helps; `clippy::let_underscore_must_use` catches silent drops. `.unwrap()` / `.expect()` are acceptable only when you can assert "this cannot fail" with rationale in the string.

### Types

- **Use fixed-width ints.** `u32`/`i64`/`u8`. `usize` is fine when indexing a slice; avoid it as a domain type.
- **Newtypes for `index`, `count`, `size`.** `struct RecordIndex(u32); struct RecordCount(u32); struct BufferSize(usize);`. Conversion is explicit — eliminates off-by-one.
- **Prefer `NonZeroU32` etc.** when zero is invalid — moves the assertion into the type.

### Memory

- **Pre-size collections at construction:** `Vec::with_capacity(N)`, `HashMap::with_capacity(N)`. Assert `len <= capacity` before `push` in hot paths.
- **Reuse buffers across calls.** Pass `&mut Vec<T>` in instead of returning `Vec<T>`. Clear, don't drop.
- **Object pools** for hot allocations. Consider `typed-arena` if you need it, but weigh against zero-dependency principle.
- **`Box<[T; N]>`** over `Vec<T>` when the size is truly fixed.

### Compound conditions

```rust
// Prefer
if idx < len {
    if !bytes.is_empty() {
        // ...
    }
}

// Over
if idx < len && !bytes.is_empty() { /* ... */ }
```

## Function shape

- **70-line limit.** Rust allows long match arms — extract them to helpers.
- **Inverse hourglass:** few params, simple return. `-> ()` beats `-> bool` beats `-> u64` beats `-> Option<u64>` beats `-> Result<u64, E>`.
- **Leaf functions pure.** Parent owns the `match` / `if`; leaves compute values.
- **Push `if`s up, `for`s down** — hoist branches to the caller, loop inside the leaf with primitive inputs.

## Ownership / aliasing

- **`&T` by default for non-Copy args >16 bytes.** Rust's borrow checker prevents accidental copies, but `T` by value still moves — pass `&T` when you don't want to transfer ownership.
- **Don't alias state.** If you find yourself `let a = &self.x; let b = &self.x;`, reconsider — one source of truth.
- **Smallest scope.** Use block scopes `{ ... }` to limit variable lifetimes. Declare `let` as late as possible.
- **In-place construction** via out-pointer is less common in Rust (the language copies/moves efficiently), but for large fixed arrays use `MaybeUninit::uninit().assume_init_mut()` + field-wise init. Usually not worth it — prefer the language idiom.

## Errors

- **Every `Result` handled.** No `let _ = result` without a comment. `?` propagates; `.unwrap_or_else(|e| ...)` handles.
- **`thiserror`** for library errors, `anyhow` for app errors — but note the dependency cost.
- **Panic = programmer error only.** `unwrap` on a parsed value from user input is wrong; on a value you just constructed from constants is fine with an `.expect("static size cannot overflow")`.

## Naming

- **`snake_case` functions/vars/modules.** `PascalCase` types. Rust already enforces this with `rustc` lints.
- **Acronyms:** `VSRState` is Rust-idiomatic when acronyms are 2-3 chars; use `VsrState` for longer ones per rustc convention. TigerStyle's preference is short acronyms uppercase — aligned.
- **Units trailing, descending significance:** `latency_ms_max: u32` not `max_latency_ms`.
- **No abbreviations:** `src`/`dst` → `source`/`target`.
- **Helper prefixes:** `read_sector` + `read_sector_callback` — but in Rust callbacks are often closures, so this shows up more as method naming.

## Calls / arguments

- **Named args via struct literals:**
  ```rust
  let config = FetchOptions {
      cache: CacheMode::Data,
      mode: AccessMode::Read,
      locality: Locality::High,
  };
  prefetch(address, config);
  ```
  Don't rely on `Default::default()` at call sites in hot or dangerous paths — spell options out.
- **Builders for complex construction.** Keep call sites readable.
- **Singletons positional:** allocator, tracer — threaded in constructors most-general to most-specific.

## Division / off-by-one

- **Explicit rounding:** `.div_euclid(n)`, `.div_ceil(n)` (Rust 1.73+), or custom `div_floor`/`div_ceil`. Avoid bare `/` for non-exact division.
- **Checked arithmetic at boundaries:** `checked_add`, `checked_mul`.
- **Typed indexes/counts** eliminate the add-one confusion.

## Async

- **Avoid `.await` points in assertion-critical sections.** TigerStyle's "run to completion" maps to "don't suspend in the middle of invariant-dependent logic." Extract the sync core, `.await` at the edges.

## Style / formatting

- **`cargo fmt`.**
- **`cargo clippy -D warnings` at strictest setting** — corresponds to "all compiler warnings, strictest setting."
- **100-column limit** (Rust default is also 100).
- **4-space indent** (Rust default).

## Skip / adapt

- `@prefetch`, `@divExact` — Rust-specific: `std::intrinsics::prefetch_read_data` is unstable; practical answer is inline asm or SIMD intrinsics when needed.
- `comptime` blocks → `const fn` + `const` evaluation.
- In-place init "viral" rule — much less relevant; Rust's move semantics and RVO handle the motivating problem.
- `*const` for >16 bytes — replaced by `&T` / `&mut T`, which the borrow checker enforces.
