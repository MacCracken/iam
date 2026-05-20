# iam — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.6.0** — shipped 2026-05-19. M5 complete: cold-start measured
at ~1.5 ms on archaemenid (M5 gate < 10 ms, 6.5× headroom),
three-point benchmark trend captured, P(-1) security audit filed
with one open finding (F-001) tracked as v1.0 freeze gate, and
the MOTD-dogfood cycle ran clean from the v0.5.0 cut through the
v0.6.0 cut across the maintainer's full set of interactive
terminal sessions. Codebase unchanged from v0.5.0 — this cut is
pure milestone closure.

**Previous**: 0.5.0 — 2026-05-19. M4 complete: ADR 0001 output-
shape contract landed and locked into executable form via 39
byte-exact tests. Renderers refactored to write-into-buffer;
driver flushes the whole report with a single syscall.

## Toolchain

- **Cyrius pin**: `6.0.1` (in `cyrius.cyml [package].cyrius`)

## Shape

Binary (`iam`). Tiny one-shot CLI: reads system facts through mihi,
prints a fixed-shape report to stdout, exits 0. The shape is six
required lines (CPU / Memory / Kernel / Host / Distro / Uptime)
plus an optional seventh (`GPU:`) when an accelerator is detected.
No flags planned for v1.0 beyond standard `--help` / `--version`.

## Source

- `src/main.cyr` — driver: opens the shared uts buffer, calls the
  mihi probes, accumulates rendered lines into a 4 KiB stack
  buffer, flushes once. Optional GPU line is inlined here (single
  probe, single line, suppress-on-zero).
- `src/display.cyr` — `iam_render` / `iam_render_buf` /
  `iam_render_kernel` line renderers (write-into-buffer, return
  bytes-written), `iam_format_bytes` (binary units, floor),
  `iam_uint_into` shared digit helper, `iam_put` / `iam_copy`
  internal append+bounds primitives.
- `src/uptime.cyr` — `iam_format_uptime` (seconds → "1d 2h 3m" with
  zero-elision + `<1m` floor).

## Output

Current sample (on archaemenid, 2026-05-19):

```
CPU:    AMD Ryzen 7 5800H with Radeon Graphics
Memory: 59 GiB
Kernel: Linux 7.0.5-arch1-1
Host:   archaemenid
Distro: Arch Linux
Uptime: 2h 10m
GPU:    AMD Radeon (PCI 0x1002:0x1638)
```

On a GPU-less host (most servers, hosted CI runners) the last line
is suppressed and the output is six lines instead of seven.

Layout contract (locked by [`docs/adr/0001-output-shape.md`](../adr/0001-output-shape.md);
freezes at v1.0):

- 8-byte label column (`<label>:` padded with spaces).
- Single value runs to EOL; no color, no escape sequences (TTY ==
  pipe).
- Missing value renders as `(unknown)` — never a stderr error,
  never a blank line.
- The GPU line is the only optional line. Six required labels
  always appear in their documented order; GPU appends in position
  7 iff an accelerator is detected.
- Exit 0 always — probe failure is signaled in `(unknown)`, not
  the exit code (login MOTD under `set -e` must not be tripped).

## Tests

- `tests/iam.tcyr` — 90 assertions:
  - Formatter logic (51): `iam_uint_into`, `iam_format_bytes`
    (boundary + floor + preview cases), `iam_format_uptime`
    (zero-elision, fresh-boot floor, cap-too-small).
  - ADR-contract byte-exact (39): per-label padding for all seven
    labels, `(unknown)` fallback paths through every renderer
    variant, full six-line + seven-line assembled output, full
    all-probes-failed degraded shape.
- `tests/iam.bcyr` — benchmark stub (`noop` micro-bench).
- `tests/iam.fcyr` — fuzz stub.

End-to-end probe wiring is verified by building the binary and
running it on the maintainer's box (archaemenid). The byte-exact
suite covers the ADR contract executable-form for synthetic inputs.

## Dependencies

Direct (declared in `cyrius.cyml`):

- stdlib (union of iam + mihi + ai-hwaccel bundle needs): string,
  fmt, alloc, io, vec, str, slice, syscalls, assert, agnosys, fs,
  tagged, process, fnptr, thread, freelist, hashmap, ct, json, bench.
- **mihi 0.7.0** — the probe library; every displayed line is a
  `mihi_*` call.
- **ai-hwaccel 2.2.6** — pulled in transitively (mihi's
  `dist/mihi.cyr` references ai-hwaccel symbols in its GPU block).
  Unused under M1+M2 — DCE drops it from the linked binary.

mihi pin floats up through 1.0; the M6 v1.0.0 gate pins to mihi
1.0.x once mihi ships its own 1.0.

## Consumers

_None yet._ iam is end-user-facing; "consumers" are user MOTD
invocations and shell login scripts.

## Benchmarks

Three-point cold-start trend captured 2026-05-19 on archaemenid
(median of three N=500 batches per build, full methodology in
[`../benchmarks.md`](../benchmarks.md)):

| Build              | Per-invocation | Lines |
| ------------------ | --------------:| -----:|
| v0.3.0 (M1+M2)     |      ~555 µs   |     6 |
| M3 @ 9df0859       |     ~1514 µs   |     7 |
| v0.5.0 (M4)        |     ~1511 µs   |     7 |

M5 < 10 ms cold-start gate met with ~6.5× headroom. GPU probe (M3)
is the dominant cost; single-flush refactor (M4) was nominal at
this scale.

## Audit

P(-1) hardening pass filed at
[`../audit/2026-05-19-audit.md`](../audit/2026-05-19-audit.md).
**Verdict**: pass with one open item.

- **F-001** (OPEN, LOW): TTY-escape sanitization on mihi-returned
  strings — gate for M6 v1.0 freeze.
- **F-002** (INFO): `strlen` on mihi cstrings is trust-dependent —
  cross-link to mihi audit, no iam-side change.

All other categories clean (bounds, exit-code discipline, no
unsafe syscalls, no env / file / network I/O).

## Next

See [`roadmap.md`](roadmap.md). With M4 in the can and three of
four M5 items shipped, the remaining roadmap is:

- **M5.5** (v0.7.0) — security + code re-audit. Web research
  against the dep tree (mihi 0.7.0, ai-hwaccel 2.2.6, cyrius
  6.0.0, stdlib modules) for 0days / CVEs. Full code re-walk
  against the 2026-05-19 audit's checklist. Refreshed audit doc
  supersedes 2026-05-19. Held until maintainer signals start.
- **F-001 mitigation** — add control-character/escape filter at
  the renderer boundary; gate on M6 v1.0 freeze. May land in
  v0.7.0 (if M5.5 work surfaces it) or v0.9.0 RC.
- **M6** — v0.9.0 RC cuts when M5.5 is clean + F-001 mitigation
  has landed. v1.0.0 cuts once mihi 1.0 ships and the output
  shape (ADR 0001 or ADR 0002, depending on reorder acceptance)
  is frozen.

Dogfood stays live across all of this — `~/.local/bin/iam` →
`build/iam`, fired from `~/.zshrc` on every interactive shell.
Future cuts auto-propagate via the symlink.
