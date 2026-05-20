# iam — Claude Code Instructions

> **Core rule**: this file is **preferences, process, and procedures** —
> durable rules that change rarely. Volatile state (current version,
> module line counts, supported backends, test counts, dep-gap status,
> consumers) lives in [`docs/development/state.md`](docs/development/state.md).
> Do not inline state here.

## Project Identity

**iam** — *pure inverse of `whoami`*. `whoami` prints who the user is; **`iam` prints what the system is.** fastfetch/neofetch-equivalent system-info display, kept whoami-simple.

- **Type**: Binary
- **License**: GPL-3.0-only
- **Language**: Cyrius (toolchain pinned in `cyrius.cyml [package].cyrius`)
- **Version**: `VERSION` at the project root is the source of truth — do not inline the number here
- **Genesis repo**: [agnosticos](https://github.com/MacCracken/agnosticos)
- **Standards**: [First-Party Standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/first-party-standards.md) · [First-Party Documentation](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/first-party-documentation.md)
- **Shared crates registry**: [shared-crates.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/shared-crates.md)

## Goal

Tell the user what their box is, in the terse register of `whoami`. A thin presentation layer over [`mihi`](https://github.com/MacCracken/mihi) (the shared probe library) — no theming engine, no plugin system, no configurable output, no ASCII-logo-of-the-month-club. Fifth member of the terminal-aesthetics set (commandress / darshini / hapi / BannerManor / **iam**).

## Current State

> Volatile state lives in [`docs/development/state.md`](docs/development/state.md) —
> current version, mihi version pin, output shape, in-flight work,
> dep gaps. Refreshed every release.

This file (`CLAUDE.md`) is durable rules.

## Scaffolding

Project was scaffolded with `cyrius init iam`. **Do not manually create project structure** — use the tools. If the tools are missing something, fix the tools.

## Quick Start

```sh
cyrius deps                          # resolve stdlib + mihi (when wired)
cyrius build src/main.cyr build/iam  # compile
./build/iam                           # prints "iam v0.1.0 — scaffold"
cyrius test                           # run tests/*.tcyr
```

## Key Principles

- **Keep it whoami-simple.** This is the load-bearing principle. `whoami` is one line. `iam` is a small handful of lines — the essential facts the box has to declare about itself. Resist every "make it more like neofetch" pull.
- **NO theming engine. NO plugin system. NO configurable output.** Same minimalism as `whoami`. If a user wants a different format, they pipe `iam` through `awk` / `cut` / `jq` like they would `whoami`.
- **NO ASCII-logo-of-the-month-club.** ASCII logos are out of scope. (Render banners with `BannerManor`, not `iam`.)
- **Probe via mihi, never inline.** `iam` is the presentation surface; mihi owns the probes. Don't reach into `/proc` from `iam` — go through the mihi API. (See *one source per fact* in mihi's CLAUDE.md.)
- **One redraw per invocation.** Login shell calls `iam`; output flushes; process exits. No daemon, no cache, no preview server.
- **Pipe-aware.** Auto-detect TTY; suppress any future color decoration when stdout isn't a terminal.

## Rules (Hard Constraints)

- **Read the genesis repo's CLAUDE.md first** — [agnosticos/CLAUDE.md](https://github.com/MacCracken/agnosticos/blob/main/CLAUDE.md)
- **Do not commit or push** — the user handles all git operations
- **Never use `gh` CLI** — use `curl` to the GitHub API only
- Do not skip tests before claiming changes work
- Do not use `sys_system()` or `exec_*` — `iam` is read-only, syscall-only
- Do not call into `/proc` / `/sys` directly — probes route through mihi; if mihi lacks a probe you need, add it to mihi first
- Do not trust external data without validation — even mihi output gets bounds-checked at display time
- Do not modify `lib/` files (vendored stdlib / dep symlinks managed by `cyrius deps`)
- Do not introduce configurability through env vars / config files / dotfiles. The output shape is the contract.
- Do not hardcode toolchain versions in CI YAML — `cyrius = "X.Y.Z"` in `cyrius.cyml` is the source of truth

## Process

### P(-1): Hardening (before v0.2.0 first feature cut, and before v1.0)

1. **Cleanliness** — `cyrius build`, `cyrius lint`, tests pass
2. **Benchmark baseline** — `cyrius bench` for the full `iam` run (this is a login-hot path, runtime matters)
3. **Internal review** — every mihi return checked, every output write bounded
4. **External research** — fastfetch / neofetch output shape conventions; learn what to deliberately *not* do
5. **Security audit** — file findings in `docs/audit/YYYY-MM-DD-audit.md`
6. **Documentation audit** — output-shape ADR (the contract that locks at v1.0)

### Work Loop (continuous)

1. **Work phase** — new line in the output, bug fix, mihi pin bump
2. **Build check** — `cyrius build src/main.cyr build/iam`
3. **Test additions** — happy + mihi-error path per displayed line
4. **Internal review** — output bounds, mihi return handling
5. **Documentation** — CHANGELOG, state.md
6. **Version sync** — `VERSION`, `cyrius.cyml`, CHANGELOG header in sync before tag

### Task Sizing

- **Low/Medium effort**: batch — multiple display lines per cycle
- **Large effort**: small bites — output-shape changes are v1.0-defining
- **If unsure**: treat it as large

### Feature-creep gate

Before adding anything to the output: ask "could this go in a sibling
tool instead?" If yes, send it there. `iam`'s value is precisely in
what it *doesn't* do.

## Cyrius Conventions

- All struct fields are 8 bytes (`i64`), accessed via `load64`/`store64` with offset
- Heap allocation via `fl_alloc()`/`fl_free()` for individual-lifetime data
- Bump allocation via `alloc()` for long-lived data
- Enum values for constants — don't consume `gvar_toks` slots
- `break` in while loops with `var` declarations is unreliable — use flag + `continue`
- See [cyrius CLAUDE.md](https://github.com/MacCracken/cyrius/blob/main/CLAUDE.md) for the full convention set

## Docs

- [`docs/adr/`](docs/adr/) — Architecture Decision Records (*why X over Y?*)
- [`docs/architecture/`](docs/architecture/) — Non-obvious constraints
- [`docs/guides/`](docs/guides/) — Task-oriented how-tos
- [`docs/examples/`](docs/examples/) — Sample outputs
- [`docs/development/state.md`](docs/development/state.md) — Live state snapshot
- [`docs/development/roadmap.md`](docs/development/roadmap.md) — Milestones through v1.0
- [`docs/doc-health.md`](docs/doc-health.md) — Doc-currency ledger (fresh / stale / evergreen / open-question per file)

Full doc-tree convention: [first-party-documentation.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/first-party-documentation.md).

## CHANGELOG Format

Follow [Keep a Changelog](https://keepachangelog.com/). Output-shape changes are `Breaking` until v1.0, after which the line order, label format, and exit codes are frozen.
