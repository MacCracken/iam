# iam — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.1.0** — scaffolded 2026-05-19 via `cyrius init iam`. No releases yet.

## Toolchain

- **Cyrius pin**: `6.0.0` (in `cyrius.cyml [package].cyrius`)

## Shape

Binary (`iam`). Tiny one-shot CLI: prints a fixed-shape system-fact
report to stdout, exits. No flags planned for v1.0 beyond the
standard `--help` / `--version`.

## Source

- `src/main.cyr` — entry point; currently prints scaffold version line

M1 onward fills:

- `src/display.cyr` — line formatter (label-padded output)
- `src/uptime.cyr` — seconds → `1d 2h 3m` formatter (M2)
- mihi consumption inlined in `main.cyr` for now; carved out if surface grows

## Output

_None yet._ Planned final shape (locked at M4 ADR; subject to
refinement through M4):

```
CPU:    <model>
Memory: <total>
Kernel: <name> <version>
Host:   <hostname>
Distro: <pretty-name>
Uptime: <human-format>
GPU:    <vendor> <model>     (suppressed if absent)
```

## Tests

- `tests/iam.tcyr` — primary suite (currently empty per cyrius init defaults)
- `tests/iam.bcyr` — benchmark stub
- `tests/iam.fcyr` — fuzz stub

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib — string, fmt, alloc, io, vec, str, syscalls, assert, bench

M1 adds `[deps.mihi]` pinned to mihi ≥ 0.2.0; v1.0 pins to mihi 1.0.x.

## Consumers

_None yet._ iam is end-user-facing; "consumers" are user MOTD
invocations and shell login scripts.

## Next

See [`roadmap.md`](roadmap.md). Next ship is M1 (pin mihi + display CPU/RAM/kernel), gated on mihi M1 landing.
