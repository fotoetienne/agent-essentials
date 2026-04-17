# TigerStyle in TypeScript

How TigerStyle translates to TypeScript. TS's type system does heavy lifting for some TigerStyle rules; others (assertion density, allocation discipline) require conscious effort in a language with a GC and an event loop.

## Safety

### Assertions

- **`assert`** from `node:assert/strict` or a tiny helper:
  ```ts
  function assert(condition: unknown, message?: string): asserts condition {
    if (!condition) throw new Error(`Assertion failed: ${message ?? ""}`);
  }

  function transfer(debit: bigint, credit: bigint, amount: bigint): void {
    assert(debit !== credit, "debit and credit must differ");
    assert(amount > 0n, `amount must be positive, got ${amount}`);
    // ...
  }
  ```
  The `asserts condition` signature narrows types — TS treats assertions as type refinements.
- **Split compound asserts:**
  ```ts
  assert(a > 0, "a > 0");
  assert(b < max, "b < max");
  // not: assert(a > 0 && b < max, "...");
  ```
- **Implications:** `if (a) assert(b, "b must hold when a");`.
- **Compile-time asserts:** via conditional types / `never` tricks:
  ```ts
  type AssertTrue<T extends true> = T;
  type _Check = AssertTrue<BufferSize extends number ? true : false>;
  ```
- **Pair assertions:** validate on serialize (`JSON.stringify` the parsed DTO and re-check shape) and on deserialize (via Zod / type guards).
- **Two per function average** — TS types cover many preconditions at the boundary; add runtime asserts for bounds, counts, and invariants the type can't express.

### Control flow

- **Bounded loops.** Prefer `for (let i = 0; i < n; i++)` and explicit limits on iterables. Be suspicious of `for await (const x of infiniteStream)` without a bound.
- **No recursion** where iteration works. JS engines have no guaranteed TCO.
- **Handle every promise and error.** `no-floating-promises` ESLint rule. Never `catch (e) {}`. Use `try/catch` or `.catch(handler)` — not both, not neither.
- **Exhaustive `switch`** on discriminated unions: `default` case should `assertNever(x)` so adding a new variant forces an update.

### Types

- **`strict: true`** in `tsconfig.json` — non-negotiable. `noUncheckedIndexedAccess: true`, `noImplicitOverride: true`, `exactOptionalPropertyTypes: true`.
- **Branded types for domain primitives:**
  ```ts
  type RecordIndex = number & { readonly __brand: "RecordIndex" };
  type RecordCount = number & { readonly __brand: "RecordCount" };
  type BufferSize  = number & { readonly __brand: "BufferSize" };

  const asRecordCount = (n: number): RecordCount => {
    assert(Number.isInteger(n) && n >= 0, `invalid count: ${n}`);
    return n as RecordCount;
  };
  ```
  Kills off-by-one — you can't pass a `RecordIndex` where a `RecordCount` is expected.
- **Use `bigint`** for values that may exceed 2^53. Don't silently lose precision.
- **No `any`.** `unknown` at boundaries, narrow explicitly. `eslint-disable` requires a comment.

### Memory

- **Pre-size arrays** where the size is known: `new Array(n)` or `Array.from({length: n})`.
- **Reuse buffers** in hot paths — pass a `Buffer` / `Uint8Array` in, don't allocate per call.
- **Avoid intermediate arrays** in hot loops: `for` loop over `[].filter().map().reduce()` — each call allocates.
- **Object pools** for extremely hot allocations (game loops, parsers). Rare in typical TS code — V8's young gen is cheap.

### Compound conditions

```ts
// Prefer
if (index < length) {
  if (bytes.length > 0) {
    // ...
  }
}

// Over
if (index < length && bytes.length > 0) { /* ... */ }
```

## Function shape

- **70-line limit.** TypeScript's verbosity (type annotations, import boilerplate) uses lines — 70 is still the right bar.
- **Inverse hourglass:** few params, simple return. Use a typed options object for >3 args:
  ```ts
  type FetchOptions = { cache: "data" | "none"; mode: "read" | "write"; locality: 0 | 1 | 2 | 3 };
  prefetch(address, { cache: "data", mode: "read", locality: 3 });
  ```
- **Leaf functions pure.** Parent owns `if`/`switch`; leaves compute values.
- **Push `if`s up, `for`s down.** Hoist branches; loop inside leaves with primitive inputs.
- **Arrow functions vs named functions:** prefer named `function` for top-level declarations — better stack traces, easier to find in devtools. Arrow functions for callbacks.

## State & aliasing

- **`const` by default.** `let` is a flag for reviewers. Never `var`.
- **`readonly` on properties and arrays.** `readonly T[]` at API boundaries prevents accidental mutation.
- **Immutable updates:** `{ ...obj, field: newValue }` or Immer, not in-place mutation of shared state.
- **Don't alias mutable state:** `const x = state.thing;` — if `state.thing` is reassigned, `x` goes stale. Snapshot explicitly and commit to the snapshot.
- **Smallest scope.** Declare `const` at point of use, not top-of-function.

## Errors

- **Every `throw` handled or deliberately propagated.** `@typescript-eslint/no-floating-promises`, `no-misused-promises`.
- **Don't catch to log:** `catch (e) { console.warn(e) }` silently drops. Either act on the error, re-throw, or comment why swallowing is correct.
- **Result types** (`type Result<T, E> = { ok: true; value: T } | { ok: false; error: E }`) force handling. Useful at API boundaries; overkill for internal code where `throw` is clearer.
- **`never` for unreachable code:** `assertNever(x: never): never { throw new Error(\`unreachable: ${x}\`); }` — catches unhandled switch cases at compile time.

### React / async specifics

- **`useEffect` with missing deps** is TigerStyle-hostile — it creates POCPOU gaps. `eslint-plugin-react-hooks/exhaustive-deps` on error.
- **Async functions in render** — don't. Derive state synchronously, defer async to events and effects with cleanup.
- **Abort signals on every fetch.** Unbounded in-flight requests violate "put a limit on everything":
  ```ts
  const ac = new AbortController();
  fetch(url, { signal: ac.signal });
  // cleanup: ac.abort()
  ```

## Naming

- **`camelCase` functions/vars, `PascalCase` types/classes, `UPPER_SNAKE` constants.** TS convention.
- **Acronyms:** TS community splits — `JSONParser`, `XMLDoc` are common; so is `JsonParser`. Follow project convention.
- **Units trailing, descending significance:** `latencyMsMax`, not `maxLatencyMs`. Adopt for new TS code.
- **No abbreviations:** `source` / `target` over `src` / `dst`.
- **Helper prefixes:** `readSector()` + `readSectorCallback()` — or `readSector` + `onReadSector` for handler naming.
- **File naming:** `kebab-case.ts` for modules, `PascalCase.tsx` for React components — follow project convention.

## Calls / arguments

- **Object args for anything > 2 parameters of similar type.** TS doesn't have true named args, but destructuring gets 90% of the way:
  ```ts
  function transfer({ debitAccount, creditAccount, amount }: TransferOptions): void { ... }
  ```
- **Don't rely on defaults at call sites for behavior-affecting options** — spell them out. Defaults for cosmetic options (separators, labels) are fine.

## Division / off-by-one

- **Explicit rounding:** `Math.floor(a / b)`, `Math.ceil(a / b)`, `Math.trunc(a / b)`. JS `/` is float division, so `5/2 = 2.5` — always choose the rounding explicitly when you want an integer.
- **Integer overflow:** JS numbers lose precision above `Number.MAX_SAFE_INTEGER` (2^53 - 1). Use `bigint` at boundaries.

## Style / formatting

- **Prettier** with project config.
- **ESLint** with `@typescript-eslint/recommended-requiring-type-checking` + `strict`.
- **`tsc --noEmit`** clean (no warnings, no errors) is the "compiler at strictest" bar.
- **100-column limit** (Prettier default is 80 — tighten to 100 if that's the team bar).
- **2-space indent** is the TS community default; TigerStyle prefers 4 but consistency with project beats the rule.
- **Braces on every `if`** unless single-line — ESLint `curly: ["error", "all"]`.

## Framework exceptions (React / Next.js / NestJS)

- React re-renders allocate heavily per render — you can't eliminate it. Use `useMemo`/`useCallback` judiciously (not everywhere — each has a cost too).
- NestJS DI does dynamic allocation at init — can't avoid.
- Server components / RSC have their own idioms around async — follow framework patterns rather than forcing TigerStyle's "run to completion".
- GraphQL resolvers: DataLoader is the batching layer — don't reinvent.

## Skip / adapt

- `@prefetch`, `@divExact` — no TS analog; V8 optimizes hot loops automatically (hidden classes, inline caching).
- In-place init via out-pointer — JS always allocates on the heap; V8's GC handles the motivating problem. Principle doesn't apply.
- `*const` for >16 bytes — TS passes objects by reference; mark with `readonly` / `Readonly<T>` for intent.
- "Zero dependencies" — nearly impossible in TS (even a basic build pipeline pulls in dozens). Adopt the spirit: justify each dep, prefer stdlib / Node built-ins.
- CamelCase struct files — TS has no analog; module naming follows project convention.
