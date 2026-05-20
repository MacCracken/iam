# iam — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.5.0** — shipped 2026-05-19. M4 ADR landed: the bytes iam emits
are now a written contract. No source changes from v0.4.0; output
shape is still six required lines (CPU / Memory / Kernel / Host /
Distro / Uptime) + optional GPU.

## Toolchain

- **Cyrius pin**: `6.0.0` (in `cyrius.cyml [package].cyrius`)

## Shape

Binary (`iam`). Tiny one-shot CLI: reads system facts through mihi,
prints a fixed-shape report to stdout, exits 0. The shape is six
required lines (CPU / Memory / Kernel / Host / Distro / Uptime)
plus an optional seventh (`GPU:`) when an accelerator is detected.
No flags planned for v1.0 beyond standard `--help` / `--version`.

## Source

- `src/main.cyr` — driver: opens the shared uts buffer, calls the
  six mihi probes, emits lines via `src/display.cyr` /
  `src/uptime.cyr` helpers.
- `src/display.cyr` — `iam_emit` / `iam_emit_buf` / `iam_emit_kernel`
  line printers, `iam_format_bytes` (binary units, floor),
  `iam_uint_into` shared digit helper.
- `src/uptime.cyr` — `iam_format_uptime` (seconds → "1d 2h 3m" with
  zero-elision + `<1m` floor).

GPU line is inlined in `main.cyr` (single probe, single line,
suppress-on-zero — too small to deserve its own module).

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

- `tests/iam.tcyr` — 51 assertions covering `iam_uint_into`,
  `iam_format_bytes` (boundary + floor + preview cases),
  `iam_format_uptime` (zero-elision, fresh-boot floor, cap-too-small).
- `tests/iam.bcyr` — benchmark stub (`noop` micro-bench).
- `tests/iam.fcyr` — fuzz stub.

End-to-end probe wiring is verified by building the binary and
running it on the maintainer's box (archaemenid). The byte-exact
output assertions land at M4 with the mihi-mock harness.

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

## Next

See [`roadmap.md`](roadmap.md). With M4 ADR in the can, the next
named milestone is:

- **M4 follow-up** (separate bite) — byte-exact tests against a
  fixed mihi-mock input. Locks the ADR contract into executable
  form. Requires routing `iam_emit` writes through a caller-buffer
  or building a mihi shim.
- **M5** — harden + dogfood pass (security audit, cold-start
  benchmark < 10ms gate, three-point benchmark trend).
- **M6** — v1.0.0 cut once mihi 1.0 ships.
