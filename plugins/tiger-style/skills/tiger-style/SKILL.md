---
name: tiger-style
description: Apply TigerBeetle's TigerStyle coding principles (safety, performance, developer experience) when writing or reviewing code. Use when the user says "TigerStyle", "tiger style", "apply tiger style", asks for a TigerStyle review, or asks about the NASA Power of Ten rules, assertion density, 70-line function limit, or batching and back-of-envelope performance sketches. Language-agnostic — translates Zig-specific rules to the language at hand.
---

# TigerStyle

TigerBeetle's coding style. Design goals, in priority order: **safety**, **performance**, **developer experience**. Source: [TIGER_STYLE.md](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md).

## First invocation: read the references

The TL;DR below is a shortlist, not the full guide. **On first invocation in a session, use the Read tool to load the reference files you'll actually need** before responding:

- **Always read** [references/principles.md](references/principles.md) — full distilled rules with rationale.
- **If reviewing** (user said review/audit/check, or pointed at a diff/PR), also read [references/review-checklist.md](references/review-checklist.md).
- **If the target language is Rust, Java, Kotlin, or TypeScript/JavaScript**, also read the matching file listed under Language Scope.

Exception: if the user asks a single narrow question that the TL;DR answers directly (e.g. "what's the function line limit in TigerStyle?"), you can answer from the TL;DR without loading references. When in doubt, read.

In subsequent turns of the same session, references are already in context — don't re-read unless you need a specific section you missed.

## Two Modes

### Reference mode — authoring code

User is writing new code or asks "how should I write this in TigerStyle?". Apply the rules from `principles.md` proactively. Cite the rule you're applying when it's non-obvious.

### Review mode — auditing changes

User says "tiger style review" or asks you to review changes against TigerStyle. Walk `review-checklist.md` against `git diff` (or the files in scope), report violations with file:line citations grouped by severity.

**Default to reference mode.** If the user says "review", "audit", "check", or points at a diff/PR, switch to review mode.

## Language Scope

TigerStyle originates in Zig, but most principles are language-agnostic. Apply cross-language rules everywhere; skip or translate Zig-only rules.

- **Zig-only rules** (skip in other languages): `zig fmt`, `@divExact`/`@divFloor`, `@prefetch` options, `usize` vs `u32`, `*const` for 16+ byte args, in-place init via out-pointer, struct field/type/method ordering idiom, the `options: struct` named-argument pattern, CamelCase struct files.
- **Translates cleanly**: assertions, function length, naming, no dynamic allocation after init (→ pre-allocated pools / object reuse), explicit control flow, error handling, comments that say *why*, batching, back-of-envelope sketches.

For language-specific translations, load the matching reference:

- **Rust** → [references/rust.md](references/rust.md)
- **Java** → [references/java.md](references/java.md)
- **Kotlin** → [references/kotlin.md](references/kotlin.md)
- **TypeScript / JavaScript** → [references/typescript.md](references/typescript.md)

For other languages (Python, Go, C++, etc.), apply the principles from [references/principles.md](references/principles.md) directly and skip the Zig-only rules listed above.

## Core Principles (TL;DR)

1. **Zero technical debt.** Do it right the first time. The second time may not transpire.
2. **Assertions are force multipliers.** Minimum two per function on average. Assert arguments, return values, pre/post-conditions, invariants. Assert both positive *and* negative space.
3. **Put a limit on everything.** Every loop, every queue, every buffer has a fixed upper bound. Fail fast.
4. **No dynamic allocation after init.** Static allocation at startup. (Translate to language idiom: object pools, pre-sized collections, arena allocators.)
5. **Hard 70-line function limit.** Push `if`s up and `for`s down. Centralize control flow in the parent; keep leaves pure.
6. **Hard 100-column line limit.** No exceptions.
7. **Explicit control flow.** No recursion where iteration works. Minimal abstractions. Split compound conditions into nested `if/else`.
8. **Handle every error.** 92% of catastrophic failures in distributed systems come from mishandled non-fatal errors.
9. **Name things well.** Units last, descending significance: `latency_ms_max`, not `max_latency_ms`. Don't abbreviate. Match character lengths of related names (`source`/`target`, not `src`/`dest`).
10. **Say why.** Comments explain rationale, not what the code does. Commit messages inform and delight.
11. **Back-of-envelope first.** Think performance at design time, across the four resources (network, disk, memory, CPU) × two dimensions (bandwidth, latency). Optimize slowest resources first, weighted by frequency.
12. **Batch everything.** Amortize costs. Don't react to external events — run at your own pace.
13. **Shrink scope.** Declare variables at the smallest possible scope; calculate close to where they're used (avoid POCPOU — place-of-check to place-of-use gaps).

Full detail in [references/principles.md](references/principles.md).

## Review-Mode Output Format

When reviewing, group findings by severity:

```
## TigerStyle Review: <target>

### P0 — Safety violations
- path/to/file.ext:L42 — Unbounded loop over `items` (no fixed upper bound). Rule: "put a limit on everything".
- path/to/file.ext:L108 — Function is 94 lines. Rule: "hard 70-line limit". Suggest extracting <specific helper>.

### P1 — Assertion gaps
- path/to/file.ext:L15 — `process()` has no precondition assertions on its 3 arguments. Rule: "assert all function arguments".

### P2 — Naming / DX
- path/to/file.ext:L7 — `maxLatencyMs` should be `latencyMsMax` (units last, descending significance).

### Notes
- Zig-only rules skipped: <list>
- What's already TigerStyle-compliant: <brief note>
```

Cite the specific TigerStyle rule for each finding. Don't pad with things that aren't violations.

## When NOT to Use

- User is asking about general code quality without invoking TigerStyle → use `simplify` skill or review normally.
- Code is in a framework-heavy context (Spring, React) where TigerStyle rules conflict with framework idioms — flag the conflict, don't force the rule. Frameworks often require dynamic allocation, recursion, and deep abstractions; note the tradeoff rather than demanding a rewrite.
- Tests with descriptive names that exceed line limits — the rule targets production code.
