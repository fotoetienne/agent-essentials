# TigerStyle Review Checklist

Use in **review mode**. Walk this list against the target (a diff, a PR, a file, a function). Skip Zig-only rules in non-Zig code — note them at the end of the report.

## Process

1. Identify scope. `git diff main...HEAD`, a PR number, or explicit files.
2. Identify language. Load the matching reference (`rust.md`, `java.md`, `kotlin.md`, `typescript.md`) if applicable.
3. Walk the checklist. Group findings by severity.
4. Output in the format shown in SKILL.md. Cite file:line and the specific TigerStyle rule for each finding.
5. Note what's already compliant — avoid only-negatives output.

## P0 — Safety violations (fix before merge)

- [ ] **Unbounded loops / queues / buffers.** Every iteration and allocation must have a fixed upper bound. Event loops must assert non-termination is intentional.
- [ ] **Dynamic allocation in hot paths / after init.** (Translate: growing collections in request handlers, unbounded channels, unconstrained recursion.)
- [ ] **Unhandled errors.** Silent catches, `_ = ...` that swallows errors without assertion, `except: pass`, unchecked `Result`/`Either`, missing `.unwrap_or_else`/`orElse` with rationale.
- [ ] **Recursion where iteration is correct.** Unbounded depth.
- [ ] **Compound boolean conditions hiding un-handled cases.** Split into nested `if/else` and check both branches.
- [ ] **`if` without matching `else`** where both positive and negative space have meaning.

## P1 — Assertion density & correctness

- [ ] **Function arguments unvalidated.** Precondition assertions on non-trivial args (bounds, non-null where the type doesn't guarantee it, invariants the caller must uphold).
- [ ] **Return values / post-conditions unasserted.** Especially at module boundaries.
- [ ] **Invariants not asserted.** Loop invariants, state-machine transitions, "this list is always sorted at this point".
- [ ] **Assertion density < 2/function average.** Measure across the diff, not each function in isolation.
- [ ] **Paired assertions missing.** Same property asserted on only one path (e.g., written but not read-verified).
- [ ] **Negative space unasserted.** Only the happy path is checked — nothing catches the bad values.
- [ ] **Compound `assert(a && b)`.** Split into `assert(a); assert(b);`.
- [ ] **Defaults relied on at library calls.** Options passed explicitly at the call site.

## P1 — Function shape

- [ ] **Functions >70 lines.** Split: centralize control flow in parent, extract pure leaves.
- [ ] **Functions with many parameters.** Inverse-hourglass shape — few params, simple return, meaty body.
- [ ] **Leaf functions branching.** Push `if`s up; leaves should be pure.
- [ ] **Control flow and state scattered across helpers.** Parent owns it.

## P2 — Naming

- [ ] **Abbreviations.** `src`/`dst`/`cfg`/`msg` when full names would be clearer.
- [ ] **Units leading instead of trailing.** `maxLatencyMs` → `latencyMsMax`. Descending significance.
- [ ] **Mismatched character lengths** on related vars that would line up in calculations.
- [ ] **Acronyms in mixed case.** `VsrState` → `VSRState`.
- [ ] **Overloaded terms.** Same word means two things in two places.
- [ ] **Participles instead of nouns** where the name will appear in docs/conversation.
- [ ] **Helpers not prefixed with caller name** when they're exclusively called from one function.
- [ ] **Callbacks not last** in parameter lists.
- [ ] **Positional args that could be mixed up.** Two same-typed args next to each other without named-args / options object.

## P2 — Scope & aliasing

- [ ] **Variables declared too early / too broadly scoped.** Move to smallest possible scope.
- [ ] **Dead variables** left around after last use.
- [ ] **Aliased variables** — `let x = self.thing.field; let y = self.thing.field;` — introduces drift risk.
- [ ] **Check far from use (POCPOU).** Calculate close to use.

## P2 — Comments & commits

- [ ] **Comments that restate the code.** Replace with a "why" or delete.
- [ ] **Missing rationale on non-obvious decisions.** "Why this threshold?" "Why this ordering?"
- [ ] **Test files without a top-of-file "goal & methodology" comment** on non-trivial tests.
- [ ] **Commit messages that describe what, not why.** (Flag, but don't block on style alone.)

## P3 — Formatting & mechanics

- [ ] **Lines >100 columns.** Hard limit.
- [ ] **`if` without braces** on multi-line bodies.
- [ ] **Formatter not run.** Check with the language's formatter (`rustfmt`, `gofmt`, `black`, `prettier`, `ktlint`, etc.).
- [ ] **Indentation other than 4 spaces** if the project doesn't already have a convention. (Don't fight project conventions.)

## Performance checklist (for hot-path code)

- [ ] **Back-of-envelope exists.** Can the author explain expected N, latency, throughput for this path?
- [ ] **Slowest resource optimized first** (network > disk > memory > CPU), weighted by call frequency.
- [ ] **Batching missed.** Per-event work that could be amortized.
- [ ] **Reacting directly to external events.** Work should run at its own pace.
- [ ] **Hot loops with struct self-access** that the compiler may not be able to register-cache. Extract to standalone fn with primitive args.
- [ ] **Control plane / data plane mixed.** Batching across the boundary lost.

## Reporting

Template:

```
## TigerStyle Review: <scope>

### P0 — Safety
- <file:line> — <specific violation>. Rule: <quote>. Fix: <specific suggestion>.

### P1 — Assertions / function shape
- <file:line> — ...

### P2 — Naming / scope / comments
- <file:line> — ...

### P3 — Formatting
- <file:line> — ...

### Compliant
- <brief note on what's already TigerStyle>

### Zig-only rules skipped
- <list of skipped rules with reason>
```

If nothing at a severity level: omit the section. Don't pad.
