# iam — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**0.8.0** — shipped 2026-05-19. ADR 0002 accepted: output-shape
reorder lands. Line order moves from the v0.3.0-era CPU-first
ordering to the identity → runtime → hardware spine (Distro / Host
/ Kernel / Uptime / CPU / GPU? / Memory), with the optional GPU
line now slotted between CPU and Memory instead of trailing at
position 7. `src/main.cyr` accumulation block reordered (probe
calls unchanged); 39 ADR-contract byte-exact tests regenerated;
`docs/examples/sample-output.txt` regenerated. **Breaking
(pre-v1.0)** by intent — done now while consumer count is zero per
audit-trail; same change post-v1.0 would have been a freeze
violation. ADR 0001 marked `Superseded by 0002`; §1, §4, §5, §6
carry over. F-001 still the lone v1.0 blocker; this is the first
source change since v0.5.0, so the M5.5 audit's *Next audit
trigger* clause is now armed for the F-001-mitigation cut.

**Previous**: 0.7.0 — 2026-05-19. M5.5 complete: dep-tree CVE / 0day
web research returned clean, full source re-walk against the M5
findings checklist confirmed zero drift since v0.5.0, refreshed
audit doc filed. Source unchanged from v0.6.0.

**Previous**: 0.6.0 — 2026-05-19. M5 complete: cold-start measured
at ~1.5 ms on archaemenid (M5 gate < 10 ms, 6.5× headroom),
three-point benchmark trend captured, P(-1) security audit filed
with one open finding (F-001) tracked as v1.0 freeze gate, and
the MOTD-dogfood cycle ran clean across the maintainer's full set
of interactive terminal sessions. Codebase unchanged from v0.5.0.

**Previous**: 0.5.0 — 2026-05-19. M4 complete: ADR 0001 output-
shape contract landed and locked into executable form via 39
byte-exact tests. Renderers refactored to write-into-buffer;
driver flushes the whole report with a single syscall.

## Toolchain

- **Cyrius pin**: `6.0.1` (in `cyrius.cyml [package].cyrius`)

## Shape

Binary (`iam`). Tiny one-shot CLI: reads system facts through mihi,
prints a fixed-shape report to stdout, exits 0. The shape is six
required lines (Distro / Host / Kernel / Uptime / CPU / Memory)
plus an optional `GPU:` line slotted between CPU and Memory when an
accelerator is detected — identity → runtime → hardware spine per
[`docs/adr/0002-output-shape-reorder.md`](../adr/0002-output-shape-reorder.md).
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
Distro: Arch Linux
Host:   archaemenid
Kernel: Linux 7.0.5-arch1-1
Uptime: 33m
CPU:    AMD Ryzen 7 5800H with Radeon Graphics
GPU:    AMD Radeon (PCI 0x1002:0x1638)
Memory: 59 GiB
```

On a GPU-less host (most servers, hosted CI runners) the GPU line
is suppressed and `Memory` slides up to position 6; the output is
six lines instead of seven.

Layout contract (locked by [`docs/adr/0002-output-shape-reorder.md`](../adr/0002-output-shape-reorder.md),
which supersedes ADR 0001 §2-§3; ADR 0001 §1, §4, §5, §6 carry over
unchanged. Freezes at v1.0):

- 8-byte label column (`<label>:` padded with spaces).
- Identity → runtime → hardware spine: Distro / Host / Kernel /
  Uptime / CPU / Memory.
- Single value runs to EOL; no color, no escape sequences (TTY ==
  pipe).
- Missing value renders as `(unknown)` — never a stderr error,
  never a blank line.
- The GPU line is the only optional line. Six required labels
  always appear in their documented order; GPU slots between CPU
  and Memory iff an accelerator is detected, keeping the hardware
  block contiguous.
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

M5.5 refreshed audit at
[`../audit/2026-05-19-m5.5-audit.md`](../audit/2026-05-19-m5.5-audit.md)
supersedes the M5 doc
([`../audit/2026-05-19-audit.md`](../audit/2026-05-19-audit.md))
for the v0.7.0 scope; the M5 doc stays in the directory as
historical record. **Verdict**: pass, no new findings.

- **F-001** (OPEN, LOW): TTY-escape sanitization on mihi-returned
  strings — gate for M6 v1.0 freeze. Carried forward from M5.
- **F-002** (INFO): `strlen` on mihi cstrings is trust-dependent —
  cross-link to mihi audit, no iam-side change. Carried forward.
- **Dep-tree CVE / 0day search**: clean. All direct deps are
  first-party MacCracken projects with no public CVE / advisory
  registrations. Cyrus IMAP / NVIDIA Container Toolkit
  name-collision noise documented and dismissed.
- **Source re-walk**: zero drift since v0.5.0 audit baseline.

All other categories clean (bounds, exit-code discipline, no
unsafe syscalls, no env / file / network I/O).

## Next

See [`roadmap.md`](roadmap.md). With ADR 0002 accepted and the
output-shape contract reordered at v0.8.0, the remaining roadmap is:

- **F-001 mitigation** — add control-character / escape filter at
  the renderer boundary in `src/display.cyr` (replace bytes
  `< 0x20` except `\n` with `?`; strip `\x1b` from mihi-returned
  value buffers inside `iam_render` / `iam_render_buf` /
  `iam_render_kernel`). Gates the M6 freeze. v0.8.0 was the first
  source change since v0.5.0 (the reorder), so the M5.5 audit's
  *Next audit trigger* clause is now armed — the F-001 cut should
  bundle a follow-up audit confirming the new output-shape code +
  the sanitizer's bounds.
- **M6** — v0.9.0 RC cuts when F-001 mitigation has landed and the
  follow-up audit is clean. v1.0.0 cuts once mihi 1.0 ships; the
  v1.0 byte contract is now the ADR 0002 shape.

Dogfood stays live across all of this — `~/.local/bin/iam` →
`build/iam`, fired from `~/.zshrc` on every interactive shell.
Future cuts auto-propagate via the symlink.
