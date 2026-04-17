# tiger-style

Apply [TigerBeetle's TigerStyle](https://tigerstyle.dev/) coding principles (safety, performance, developer experience) when writing or reviewing code.

Source: [tigerstyle.dev](https://tigerstyle.dev/) · [TIGER_STYLE.md on GitHub](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md)

## What it does

**Reference mode** (default) — while authoring code, cite TigerStyle rules (NASA Power of Ten, assertion density, 70-line function limit, naming conventions, back-of-envelope performance sketches) as they apply.

**Review mode** — walk a P0-P3 severity checklist against a diff or PR, reporting violations with `file:line` citations and the specific rule being cited.

Triggered by phrases like:
- "apply tiger style"
- "tiger style review"
- Questions about the 70-line function limit, assertion density, NASA Power of Ten, etc.

## Language support

- **Cross-language principles** (assertions, function shape, naming, error handling, batching) — applied in any language.
- **Zig-specific rules** (`zig fmt`, `@divExact`, `@prefetch`, in-place init via out-pointer, CamelCase struct files) — skipped in other languages.
- **Dedicated guides** for Rust, Java, Kotlin, TypeScript/JavaScript — auto-loaded when the target language matches.

## Framework awareness

Flags conflicts between TigerStyle and Spring / JPA / React / NestJS / DGS idioms rather than forcing rewrites. Example: `@Autowired` does dynamic allocation — that's a framework requirement, not a TigerStyle violation to fix.

## Layout

```
skills/tiger-style/
├── SKILL.md                       # skill entry, triggers, TL;DR
└── references/
    ├── principles.md              # full distilled guide
    ├── review-checklist.md        # P0-P3 audit checklist
    ├── rust.md                    # Rust translation
    ├── java.md                    # Java translation (Spring/DGS-aware)
    ├── kotlin.md                  # Kotlin translation
    └── typescript.md              # TS/JS translation (React-aware)
```

## Install

From this marketplace:

```
/plugin install tiger-style@agent-essentials
```

## Credit

Principles are TigerBeetle's — see [tigerstyle.dev](https://tigerstyle.dev/) or the canonical [TIGER_STYLE.md](https://github.com/tigerbeetle/tigerbeetle/blob/main/docs/TIGER_STYLE.md) in the TigerBeetle repo. This plugin is a structured adaptation for Claude Code, not the original guide.
