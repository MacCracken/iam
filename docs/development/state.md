# iam — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.3.0** — shipped 2026-05-19. M1+M2 collapsed into one cut: six
labeled lines (CPU / Memory / Kernel / Host / Distro / Uptime),
every value sourced from mihi.

## Toolchain

- **Cyrius pin**: `6.0.0` (in `cyrius.cyml [package].cyrius`)

## Shape

Binary (`iam`). Tiny one-shot CLI: reads system facts through mihi,
prints a fixed-shape six-line report to stdout, exits 0. No flags
planned for v1.0 beyond standard `--help` / `--version`.

## Source

- `src/main.cyr` — driver: opens the shared uts buffer, calls the
  six mihi probes, emits lines via `src/display.cyr` /
  `src/uptime.cyr` helpers.
- `src/display.cyr` — `iam_emit` / `iam_emit_buf` / `iam_emit_kernel`
  line printers, `iam_format_bytes` (binary units, floor),
  `iam_uint_into` shared digit helper.
- `src/uptime.cyr` — `iam_format_uptime` (seconds → "1d 2h 3m" with
  zero-elision + `<1m` floor).

Pending modules:

- M3 will add a `GPU:` line driven by `mihi_gpu_*`. Likely stays
  inlined in `main.cyr` — single probe, single line.

## Output

Current sample (on archaemenid, 2026-05-19):

```
CPU:    AMD Ryzen 7 5800H with Radeon Graphics
Memory: 59 GiB
Kernel: Linux 7.0.5-arch1-1
Host:   archaemenid
Distro: Arch Linux
Uptime: 1h 58m
```

Layout contract (locks at M4 ADR; subject to refinement until then):

- 8-byte label column (`<label>:` padded with spaces).
- Single value runs to EOL; no color, no escape sequences (TTY ==
  pipe).
- Missing value renders as `(unknown)` — never a stderr error,
  never a blank line.

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

See [`roadmap.md`](roadmap.md). With M1+M2 in the can, the next
named milestones are:

- **M3** — `GPU:` line driven by `mihi_gpu_*` (mihi already at the
  required version).
- **M4** — output-shape ADR locking line order, label format,
  separator, exit codes. v1.0 contract precursor.
