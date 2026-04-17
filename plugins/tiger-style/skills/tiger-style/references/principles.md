# TigerStyle Principles

Distilled from [TIGER_STYLE.md](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md). Organized by design goal.

## The Philosophy

- **Style = design.** Design goals in priority order: safety, performance, developer experience.
- **Simplicity is elegance, not concession.** It's the hardest revision, not the first attempt.
- **Zero technical debt.** Code, like steel, is cheaper to change while hot. A problem solved in production is many times more expensive than one solved in design.
- **Spend the mental energy upfront.** An hour or day of design is worth weeks or months in production.

---

## Safety

Built on NASA's *Power of Ten — Rules for Developing Safety Critical Code*.

### Control flow

- **Only simple, explicit control flow.** No recursion where iteration works — ensures bounded executions are bounded.
- **Minimal abstractions.** Abstractions are [never zero cost](https://isaacfreund.com/blog/2022-05/). Every abstraction risks leakiness.
- **Put a limit on everything.** All loops and queues need a fixed upper bound. Where a loop genuinely cannot terminate (event loop), assert that.
- **Don't react to external events directly.** Run at your own pace — batch instead of context-switching per event. Easier to bound work-per-period.

### Types

- **Explicitly-sized types.** `u32` / `i64` / `uint32_t`. Avoid architecture-specific `usize`/`size_t` where not required. *(Zig-origin rule; applies wherever the language offers a choice.)*

### Assertions

Assertions detect *programmer* errors. Operating errors are expected and handled; assertion failures are unexpected — the only correct response is to crash. Assertions downgrade catastrophic correctness bugs into liveness bugs.

- **Assert all function arguments, return values, pre/post-conditions, invariants.** Minimum **two assertions per function on average**.
- **Pair assertions.** For every property, find at least two code paths to assert it. Example: validate data right before writing to disk *and* immediately after reading from disk.
- **Assert positive and negative space.** Assert what you expect AND what you don't. Interesting bugs live at the valid/invalid boundary.
- **Split compound assertions.** Prefer `assert(a); assert(b);` over `assert(a && b)` — simpler to read, more precise when it fails.
- **Use single-line `if` to assert implications.** `if (a) assert(b);`.
- **Assert relationships of compile-time constants.** Checks program integrity before it even runs. *(Language-dependent: Zig `comptime`, Rust `const` asserts, Java static init blocks, C++ `static_assert`.)*
- **Occasionally use blatantly-true assertions as documentation** where the invariant is critical and surprising.
- **Assertions aren't a substitute for understanding.** Build a precise mental model first, *then* encode it as assertions.

### Memory

- **All memory statically allocated at startup.** No dynamic allocation or realloc after init. Avoids unpredictable latency and use-after-free. Second-order: forces you to reason about worst-case memory upfront.
- *(Translate to language: object pools, pre-sized arrays, arena allocators, `array.reserve(N)`. See language-map.md.)*

### Scope & variables

- **Smallest possible scope.** Minimize the number of variables in scope at any point.
- **Don't introduce variables before they're needed.** Don't leave them around after.
- **POCPOU** (place-of-check to place-of-use) — the cousin of TOCTOU. Calculate/check close to use.
- **Don't duplicate variables or take aliases.** Reduces the chance state drifts out of sync.

### Functions

- **Hard 70-line function limit.** Physical reason: functions that fit on a screen vs. scroll.
- **Good function shape is inverse hourglass.** Few parameters, simple return type, meaty body.
- **Centralize control flow.** ["Push `if`s up and `for`s down."](https://matklad.github.io/2023/11/15/push-ifs-up-and-fors-down.html) One function owns the branching; leaves are pure.
- **Centralize state manipulation.** Parent keeps state in locals; helpers compute *what needs to change*, parent applies it.
- **Simple return types reduce dimensionality.** `void` > `bool` > `u64` > `?u64` > `!u64`. This propagates virally through call chains.

### Errors

- **Handle every error.** [92% of catastrophic failures](https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-yuan.pdf) in distributed systems come from mishandled non-fatal errors.
- **Explicit control flow for errors too.** No silent swallowing.

### Conditionals

- **Split compound conditions** into nested `if/else`. Makes branches and cases clear.
- **State invariants positively.** `if (index < length)` — the invariant holds — is easier to reason about than `if (index >= length)`.
- **Consider whether every `if` needs a matching `else`** to handle positive and negative space.
- **Add braces to `if`** unless it fits on a single line. Defense against "goto fail;" bugs.

### Calls

- **Explicitly pass options at the call site.** Don't rely on library defaults — they can change. Example: `@prefetch(a, .{ .cache = .data, .rw = .read, .locality = 3 })` over `@prefetch(a, .{})`.

### Compiler

- **All compiler warnings, strictest setting.** From day one.

### Motivation

- **Always say why.** Rationale shares the criteria for the decision, so reviewers can evaluate edge cases.

---

## Performance

> "The lack of back-of-the-envelope performance sketches is the root of all evil."

- **Think about performance from the outset.** The 1000x wins come in the design phase — precisely when you can't measure.
- **Back-of-envelope sketches.** Four resources: network, disk, memory, CPU. Two dimensions: bandwidth, latency. Aim to land within 90% of the global maximum.
- **Optimize slowest resources first** (network → disk → memory → CPU), weighted by frequency. A memory cache miss *called many times* may cost more than a single `fsync`.
- **Distinguish control plane from data plane.** Batching across the boundary gives assertion safety without sacrificing throughput.
- **Amortize costs by batching.** Network, disk, memory, CPU.
- **Let the CPU be a sprinter.** Predictable, large chunks of work. Don't force it to zig-zag. (Again: batching.)
- **Be explicit; minimize compiler magic.** Extract hot loops into standalone functions with primitive arguments (no `self`). Compiler doesn't need to prove it can cache struct fields in registers; humans can spot redundant work.

---

## Developer Experience

### Naming

- **Get nouns and verbs right.** Names capture what a thing is or does. Take time.
- **`snake_case`** for functions, variables, files. Underscore is the programmer's space.
- **No abbreviations** unless it's a primitive integer in a sort/matrix context. Long-form CLI flags (`--force`), not single letters (those are for humans).
- **Proper capitalization for acronyms** in camel/pascal contexts: `VSRState`, not `VsrState`.
- **Units last, descending significance.** `latency_ms_max`, not `max_latency_ms`. Groups all latency variables together; lines up with `latency_ms_min`.
- **Infuse names with meaning.** `gpa: Allocator` and `arena: Allocator` beat `allocator: Allocator` — they tell the reader whether `deinit` needs to be called.
- **Same character count for related names.** `source`/`target` beats `src`/`dest` — derived names (`source_offset`, `target_offset`) line up in calculations.
- **Prefix helpers with the caller's name.** `read_sector()` → `read_sector_callback()`.
- **Callbacks go last** in the parameter list. Mirrors control flow — they're invoked last.
- **Order matters for readability.** Files are read top-down; put important things near the top. `main` goes first. In structs: fields → types → methods.
- **Don't overload names.** A single term shouldn't mean two things in two contexts. ("Two-phase commit transfers" overloaded the consensus protocol term — renamed to "pending transfers".)
- **Nouns compose better than participles.** `replica.pipeline` can be used as a section header; `replica.preparing` must be rephrased.
- **Named arguments when mixable.** Two `u64`s must use an options struct. Nullable args should be named so `null` at the call site is clear.
- **Singletons threaded positionally** from most-general to most-specific (allocator, tracer, etc).

### Comments & commits

- **Descriptive commit messages.** They're read. PR descriptions aren't in `git blame`.
- **Say why, not what.** Code alone isn't documentation.
- **Say how for tests.** Top-of-test comment describing goal and methodology so the reader can skim.
- **Comments are prose.** Space after slash, capital letter, full stop (or colon if introducing something). End-of-line comments can be phrases with no punctuation.

### Cache invalidation (avoiding stale state)

- **Don't alias or duplicate variables.** Reduces desync risk.
- **Pass large structs as `const*`** (or the language's equivalent) when you don't mean a copy. *(Zig: `*const` for >16 bytes. C++: `const T&`. Rust: `&T`. Java/Kotlin/Python: pass-by-reference is the default; the concern doesn't map 1:1.)*
- **Construct large structs in-place via out-pointer.** Avoids stack-growing intermediate copies. "In-place initialization is viral" — one in-place field means the container initializes in-place too. *(Zig-specific idiom.)*
- **Shrink scope.** Minimize variables in play at any time.
- **Calculate close to use.** Don't introduce variables before needed; don't leave them after. Avoid POCPOU.
- **Functions run to completion without suspending.** So precondition assertions hold for the function's lifetime. *(Maps to: avoid `async`/`await` in the middle of assertion-heavy logic; keep critical sections synchronous.)*
- **Beware buffer bleeds.** Heartbleed-style — not overflow, *underflow* with uncleared padding. Leaks info and breaks determinism.
- **Newline-group resource allocation and its matching `defer`/cleanup.** Makes leaks visible.

### Off-by-one

- **`index`, `count`, and `size` are distinct types.** Primitive integers with clear casting rules.
  - `index → count`: add 1 (indexes are 0-based, counts are 1-based).
  - `count → size`: multiply by unit.
  - This is why units-in-names matters.
- **Show intent in division.** `@divExact`, `@divFloor`, `div_ceil` — tell the reader you thought about rounding. *(Zig-specific names; apply the principle: use named variants over bare `/` in languages with them, or comment the intent.)*

### Style by the numbers

- **4 spaces of indentation**, not 2. More obvious at a distance.
- **Hard 100-column line limit.** Fits two copies of the code side-by-side. No horizontal scrollbars.
- **Braces on every `if`** unless it fits on one line.
- **Run the formatter.** *(Zig: `zig fmt`. Translate: `prettier`, `black`, `gofmt`, `rustfmt`, `ktlint`, etc.)*

### Dependencies

- **Zero dependencies policy** (apart from the toolchain). Supply chain, safety, performance, install time — dependencies compound costs throughout the stack.
- *(Outside TigerBeetle's domain this is extreme; the principle: each dependency has a real cost — justify it.)*

### Tooling

- **Small standardized toolbox.** One tool known well beats many tools known poorly.
- **Scripts in the project's primary language**, not shell. Cross-platform, type-safe, higher chance of working on every teammate's machine. *(TigerBeetle: `scripts/*.zig`. Translate to your stack's norm.)*

---

## The Last Stage

> "Keep trying things out, have fun, and remember—it's called TigerBeetle, not only because it's fast, but because it's small!"
