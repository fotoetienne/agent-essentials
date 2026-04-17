# TigerStyle in Kotlin

How TigerStyle translates to Kotlin. Kotlin's null safety, immutability defaults, and data classes already align with several TigerStyle rules — the translation is smoother than Java.

## Safety

### Assertions

- **`require`, `check`, `requireNotNull`, `checkNotNull`** are Kotlin's idiomatic assertion primitives.
  - `require(...)` — caller's contract (precondition on arguments).
  - `check(...)` — internal state (invariant / postcondition).
  - Both throw on failure, so they work in production (unlike `assert`, which is opt-in via `-ea`).
  ```kotlin
  fun transfer(debit: Long, credit: Long, amount: Long) {
      require(debit != credit) { "debit and credit must differ" }
      require(amount > 0) { "amount must be positive, got $amount" }
      check(balance >= amount) { "insufficient balance" }
      // ...
  }
  ```
- **Split compound asserts:**
  ```kotlin
  require(a > 0) { "a > 0" }
  require(b < max) { "b < max" }
  // not: require(a > 0 && b < max) { "..." }
  ```
- **Implications:** `if (a) require(b) { "b must hold when a" }`.
- **Compile-time asserts:** via `const val` and `init {}` blocks on `object`s.
- **Pair assertions:** round-trip on serialize/deserialize, validate at both persistence boundaries.
- **Two per function average.** Kotlin's nullable types carry a lot of what Zig asserts, but still encode non-type invariants.

### Control flow

- **Bounded iteration.** `for (i in 0 until n)`, `.take(n)` on sequences. Avoid unbounded `while (channel.receive() != null)` without a limit.
- **No recursion** for non-trivially bounded depth. Kotlin has `tailrec` for tail-recursive — prefer iteration anyway for readability.
- **Handle every result.** Kotlin's `Result<T>` / sealed error hierarchies — don't `.getOrNull()` without acting on `null`. `_ = something()` is not an idiom; use `.also { }` or explicit unused return.

### Types

- **Domain primitives as value classes:**
  ```kotlin
  @JvmInline value class RecordIndex(val value: Int)
  @JvmInline value class RecordCount(val value: Int)
  @JvmInline value class BufferSize(val value: Int)
  ```
  Zero runtime cost, explicit conversion — kills off-by-one.
- **Prefer non-nullable types.** If `null` is valid, name the variable to make it clear: `acknowledgedAtOrNull: Instant?`.
- **Sealed classes for exhaustive state machines.** The compiler enforces handling every case.

### Memory

- **Pre-size collections:** `ArrayList<T>(n)`, `HashMap<K, V>(n)`, `buildList(n) { ... }`.
- **Reuse buffers.** Pass `MutableList<T>` as a parameter and `.clear()` rather than allocating.
- **Avoid gratuitous allocations in hot paths:** `sequence { }` over `list.map { }.filter { }` chains, `buildString { }` over `+` concatenation.
- **Data class `copy()` allocates** — don't use in hot paths; mutate in place with a mutable variant.

### Compound conditions

```kotlin
// Prefer
if (index < length) {
    if (bytes.isNotEmpty()) {
        // ...
    }
}

// Over
if (index < length && bytes.isNotEmpty()) { /* ... */ }
```

Kotlin's `when` makes exhaustive branching easy — use it with sealed classes so the compiler enforces positive/negative space.

## Function shape

- **70-line limit.** Kotlin is more concise than Java, so 70 lines stretches further — but still enforce.
- **Inverse hourglass:** few params, simple return. Use data classes / records for >3 args.
- **Leaf functions pure.** Parent owns branching.
- **Push `if`s up, `for`s down.** Hoist branches to callers, loop inside leaves with primitives.
- **Extension functions** can help split responsibilities cleanly. Keep them pure and small.
- **Don't overuse `apply` / `also` / `let` / `run`.** They can obscure control flow. `apply` for configuration blocks is fine; `let` in the middle of a 40-line method isn't.

## State & aliasing

- **`val` by default.** Kotlin encourages immutability — use it. `var` is a flag for reviewers.
- **Data classes for value types.** Immutability, `equals`/`hashCode`/`copy` for free.
- **Don't alias nullable state:**
  ```kotlin
  // Risky
  val x = state.thing
  if (x != null) { x.field... }  // x could go stale if state.thing is reassigned
  ```
  Use `state.thing?.let { ... }` or a local snapshot you commit to.
- **Smallest scope.** `val` declarations can go anywhere, not just top-of-function. Declare at point of use.

## Errors

- **Every exception / `Result` handled.** Kotlin doesn't have checked exceptions — discipline is on you.
- **`runCatching`** returns `Result<T>` — still need to `.onFailure` / `.getOrElse`, don't drop.
- **Don't swallow:** `catch (e: Exception) { log.warn(e) }` without a return action is suspect. Comment why it's safe or re-throw.

## Naming

- **`camelCase` functions/vars/properties, `PascalCase` types, `UPPER_SNAKE_CASE` top-level constants.** Kotlin convention.
- **Acronyms:** Kotlin official style says treat long acronyms as words (`XmlParser`), short acronyms uppercase (`IOException`). TigerStyle's `VSRState` fits this.
- **Units trailing, descending significance:** `latencyMsMax`, not `maxLatencyMs`. Adopt for new Kotlin code.
- **No abbreviations:** `source` / `target` over `src` / `dst`.
- **Helper prefixes:** `readSector()` + `readSectorCallback()` — in Kotlin, callbacks are typically lambda params. Principle applies to named private functions.

## Calls / arguments

- **Named args are first-class in Kotlin — use them for all non-obvious call sites.**
  ```kotlin
  prefetch(address, cache = CacheMode.Data, mode = AccessMode.Read, locality = 3)
  ```
  Don't rely on positional ordering for > 1 argument of the same type.
- **Default arguments** are safe in Kotlin — but TigerStyle's "spell options at call site" still applies for behavior-affecting options; let only cosmetic defaults stand.

## Division / off-by-one

- **`Math.floorDiv(a, b)`, `Math.ceilDiv(a, b)`**, or custom `divCeil` extensions. Don't leave bare `/` for non-exact integer math — comment intent.
- **`Math.addExact` / `multiplyExact`** at boundaries for overflow-checked arithmetic. Kotlin doesn't have operator overloads for these.

## Coroutines / async

- **`suspend` functions should preserve invariants across suspension points.** TigerStyle's "run to completion" translates to: don't let a coroutine suspend in the middle of an invariant-heavy section. Use `withContext(NonCancellable)` or restructure to keep the critical section synchronous.
- **Structured concurrency by default.** `coroutineScope { ... }` — every child cancels if parent fails. Matches TigerStyle's "handle every error".
- **Flow backpressure:** `.buffer(n)` with explicit `n`, not unbounded. Aligns with "put a limit on everything".

## Framework exceptions (Spring / DGS / Ktor)

- Spring `@Autowired` / DI: dynamic allocation at init — can't avoid.
- DGS `@DgsQuery` methods: extract helpers but don't split artificially for the 70-line limit.
- Compose UI: recomposition patterns need mutable state — doesn't fit "val by default" strictly. Use `remember { mutableStateOf(...) }` and keep invariants in derived state.
- Reactive (Mono/Flux): batching and back-pressure are built in — don't reinvent.

## Style / formatting

- **ktlint / ktfmt / detekt** with strict config.
- **Compiler at strictest:** `-Werror`, `-Xopt-in=kotlin.RequiresOptIn`.
- **100-column limit** (ktlint default is 120 — tighten to 100 per TigerStyle).
- **4-space indent** (Kotlin official default).
- **Braces on every `if`** unless single-line.

## Skip / adapt

- `@prefetch`, `@divExact` — no direct Kotlin analog; use JVM intrinsics if needed.
- In-place init via out-pointer — JVM always heap-allocates; principle doesn't apply.
- `*const` for >16 bytes — Kotlin passes objects by reference. Use `val` fields in data classes for effective immutability.
- "Zero dependencies" — adopt the spirit: justify each dependency, prefer stdlib over external libs.
