# TigerStyle in Java

How TigerStyle translates to Java. Java's runtime, GC, and framework conventions (Spring, Jakarta EE) push back on several TigerStyle rules — flag the conflict rather than forcing a rewrite in framework code.

## Safety

### Assertions

- **`assert` keyword** requires `-ea` at runtime — useful in tests, often disabled in prod. For production-critical invariants:
  ```java
  if (x < 0) throw new IllegalStateException("x must be non-negative, got " + x);
  Objects.requireNonNull(arg, "arg");
  ```
  Guava's `Preconditions.checkArgument` / `checkState` is the common idiom (but weigh dependency cost).
- **Split compound asserts:**
  ```java
  Preconditions.checkArgument(a > 0, "a must be positive");
  Preconditions.checkArgument(b < max, "b must be < max");
  // not: checkArgument(a > 0 && b < max, ...)
  ```
- **Implications:** `if (a) { assert b : "b must hold when a"; }`.
- **Compile-time asserts:** limited — can use annotation processors, or put invariants in `static` blocks that throw.
- **Pair assertions:** validate DTOs on both serialize and deserialize, check invariants on both write and read paths.
- **Two per method average** — lean on `@NonNull`/`@Nullable` annotations + `Objects.requireNonNull` at entry points.

### Control flow

- **Bounded loops.** Prefer `for (int i = 0; i < N; i++)` and explicit limits over `while (iterator.hasNext())` unbounded iteration. For streams, add `.limit(N)` defensively where appropriate.
- **No recursion** for non-trivially bounded depth — Java has no TCO. Prefer iteration or explicit stack-backed traversal.
- **Handle every checked exception.** Don't `catch (Exception e) { log.warn(...); }` silently — either re-throw, act on it, or add a comment explaining why swallowing is correct.
- **`@SuppressWarnings`** requires a justification comment.

### Types

- **Use sized primitives.** `int`, `long`, `byte`. Java already enforces this.
- **Value objects for domain primitives.** `record RecordIndex(int value) {}`, `record RecordCount(int value) {}`. Explicit conversion via `.value()` eliminates off-by-one.
- **`Optional`** at method boundaries where `null` would be ambiguous — but avoid in fields and in hot paths (allocates).

### Memory

- **Pre-size collections:** `new ArrayList<>(N)`, `new HashMap<>(N)`. In hot paths assert `size <= capacity` before `add`.
- **Reuse buffers.** Pass `List<T>` / `StringBuilder` as a parameter and `.clear()` rather than allocating per-call.
- **Object pools** for very hot allocations (`ThreadLocal<byte[]>`). Most Java code doesn't need this — `young gen` GC is cheap — but TigerStyle's "no dynamic allocation after init" in a hot path is real advice for low-latency code.
- **`EnumMap` / `EnumSet`** over `HashMap`/`HashSet` when keys are enums — no allocation, no hashing.

### Compound conditions

```java
// Prefer
if (index < length) {
    if (!bytes.isEmpty()) {
        // ...
    }
}

// Over
if (index < length && !bytes.isEmpty()) { /* ... */ }
```

## Method shape

- **70-line limit.** Java gets verbose — extract helpers aggressively. A 70-line method in Java is less "work" than in Zig, so the limit bites faster.
- **Inverse hourglass:** few params, simple return. Use records or parameter objects for >3 args.
- **Leaf methods pure.** Parent owns branching; leaves compute values.
- **Push `if`s up, `for`s down.** Hoist branches to callers; loop inside the leaf with primitive inputs.
- **Constructor + factory:** keep constructors narrow; use static factories (`MyThing.of(...)`) with named semantics for alternate construction paths.

## State & aliasing

- **No aliasing to mutable state.** Don't grab `var x = this.field; var y = this.field;` — one source of truth.
- **Make fields `final` by default.** Immutability is the best defense against state drift. Use records for value types.
- **Defensive copies at API boundaries** for mutable collections: `List.copyOf(input)`, `Map.copyOf(input)`.
- **Smallest scope.** Declare `var`/`final` as late as possible. Java allows late declarations — use it.

## Errors

- **Every exception handled.** Checked exceptions force it; unchecked need discipline.
- **Don't wrap without context:** `throw new MyException("failed to write record " + id, e);` — include identifying info.
- **`IllegalArgumentException`** for caller's fault, `IllegalStateException` for internal invariant failure, domain exceptions for business errors. Crash-on-programmer-error means these should fail fast, not be caught and ignored.

## Naming

- **`camelCase` methods/vars, `PascalCase` types, `UPPER_SNAKE` constants.** Java convention — TigerStyle's `snake_case` doesn't apply in Java idioms.
- **Acronyms:** TigerStyle says `VSRState` (all caps for short acronyms). Java convention is split: `JDBC`, `HTTP` uppercase; but `HttpClient`, `XmlParser`. Follow the project's existing convention — consistency beats the rule.
- **Units trailing, descending significance:** `latencyMsMax`, not `maxLatencyMs`. This *does* apply in Java — adopt it for new code.
- **No abbreviations:** `source` / `target` over `src` / `dst`. Java already tends toward descriptive names.
- **Helper prefixes:** `readSector()` + `readSectorCallback()`. Lambda callbacks are more common in modern Java — the principle still applies to named private methods.

## Calls / arguments

- **Named args via parameter objects / builders.** Two `long` params next to each other must use a record or builder:
  ```java
  public record TransferRequest(long debitAccount, long creditAccount, long amount) {}
  transfer(new TransferRequest(debit, credit, amount));
  ```
- **Don't rely on varargs defaults** — spell out options at call sites when the arg has behavior implications.

## Division / off-by-one

- **Explicit rounding:** `Math.floorDiv(a, b)`, `Math.ceilDiv(a, b)` (Java 18+), or custom. Avoid bare `/` for non-exact integer division — comment the intent.
- **`Math.addExact` / `Math.multiplyExact`** at boundaries for overflow-checked arithmetic.

## Concurrency / async

- **Run to completion, no suspension in critical sections.** CompletableFuture chains should keep invariant-heavy logic in one stage, not split across `.thenCompose` boundaries.
- **Virtual threads (Java 21+)** preserve sync-style invariants across blocking calls — use them in preference to reactive chains when invariants matter.

## Framework exceptions (Spring / Jakarta / DGS)

**When TigerStyle conflicts with framework idioms, note the conflict and follow the framework.**
- Spring `@Autowired` does dynamic allocation — can't avoid in a Spring app.
- JPA entities require mutable fields and no-arg constructors — TigerStyle's "no aliasing" and "final by default" don't fit.
- DGS `@DgsQuery` methods can be longer than 70 lines if they're driving schema resolvers — extract helpers but don't split just to satisfy the limit.
- GraphQL resolvers often can't "batch" in the TigerStyle sense — DataLoader *is* the batching layer; don't reinvent it.

## Style / formatting

- **Spotless / google-java-format / palantir-java-format** — run on every save.
- **Compiler at strictest:** `-Xlint:all -Werror`. Lombok warnings on; `@SuppressWarnings` requires justification.
- **100-column limit** — aligns with google-java-format's 100. (Check project's setting.)
- **4-space indent** is google-java-format default.
- **Braces on every `if`.** Checkstyle rule: `NeedBraces`.

## Skip / adapt

- `@prefetch`, `@divExact` — no direct analog.
- In-place init via out-pointer — Java objects are always heap-allocated; principle doesn't apply. GC handles the motivating problem.
- `*const` for >16 bytes — Java passes objects by reference; for primitives pass records and mark fields `final`. Immutability replaces the `const` concept.
- "Zero dependencies" — extreme in Java. Adopt the spirit: justify each dependency, prefer JDK over external libs.
