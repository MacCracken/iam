# iam — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**1.0.0** — shipped 2026-05-20. **M6 closed. Output-shape contract
frozen.** Lockstep release with mihi 1.0.0. The cut is the
documented single-line `[deps.mihi] tag` bump from 0.7.0 → 1.0.0
plus milestone-closure docs/audit work; **zero `src/*.cyr` changes
since v0.9.0 RC**. mihi 1.0.0's bundle is module-content
byte-identical to 0.7.0 (per mihi's CHANGELOG: "module content
byte-identical to 0.8.0 (which was itself byte-identical to 0.7.0)"
— only the `# Version:` header stamp differs), so iam's runtime
output is byte-for-byte equal to the v0.9.0 RC baseline on
archaemenid. Mandatory mihi-major-bump audit filed at
[`../audit/2026-05-20-v1.0.0-audit.md`](../audit/2026-05-20-v1.0.0-audit.md):
pass, no new findings; F-001 stays closed, F-002 carries as INFO
(now formally version-pinned by mihi's API freeze). ADR 0002's
identity → runtime → hardware spine is the written contract from
this release onward; any future change to line order, label width,
label spelling, `(unknown)` fallback, or exit-code discipline is a
real `Breaking` requiring a major-version bump.

**Previous**: 0.9.0 — 2026-05-19. **M6 release candidate.** F-001
TTY-escape sanitizer lands as the v1.0 freeze prerequisite:
`iam_copy_value` in `src/display.cyr` replaces any value-column
byte `< 0x20` with `?`, wired into `iam_render` / `iam_render_buf`
/ `iam_render_kernel`. 15 new byte-exact assertions cover the
sanitizer surface (boundary, ESC, TAB, newline-injection, mixed
control bytes); test count 90 → 105. Bench refreshed: **1510 µs
median, 3 trials × N=500** on archaemenid — inside the v0.5.0
noise floor, sanitizer is invisible at this scale. v0.9.0 audit
([`../audit/2026-05-19-v0.9.0-audit.md`](../audit/2026-05-19-v0.9.0-audit.md))
superseded M5.5 for the F-001 scope.

**Previous**: 0.8.0 — 2026-05-19. ADR 0002 accepted: output-shape
reorder to identity → runtime → hardware spine
(Distro / Host / Kernel / Uptime / CPU / GPU? / Memory).
**Breaking (pre-v1.0)** by intent — landed while consumer count
was zero. ADR 0001 marked Superseded by 0002 (§1, §4, §5, §6 carry
over); 39 ADR-contract tests regenerated; sample-output regenerated.
First source change since v0.5.0 — armed the M5.5 audit's *Next
audit trigger* clause for the F-001 cut that followed at v0.9.0.

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

- `tests/iam.tcyr` — 105 assertions:
  - Formatter logic (51): `iam_uint_into`, `iam_format_bytes`
    (boundary + floor + preview cases), `iam_format_uptime`
    (zero-elision, fresh-boot floor, cap-too-small).
  - ADR-contract byte-exact (39): per-label padding for all seven
    labels, `(unknown)` fallback paths through every renderer
    variant, full six-line + seven-line assembled output, full
    all-probes-failed degraded shape. Regenerated at v0.8.0
    against the ADR 0002 order.
  - F-001 TTY-escape sanitization byte-exact (15): `iam_copy_value`
    boundary (0x1F / 0x20 / TAB); ESC in `iam_render` value;
    newline-injection attempt; ESC mid-buffer in `iam_render_buf`;
    ESC + TAB in `iam_render_kernel` name + version; unchanged
    `IAM_UNKNOWN_TEXT` fallback path.
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
- **mihi 1.0.0** — the probe library; every displayed line is a
  `mihi_*` call. Pinned at the freeze; the surface iam links is
  contract-bound per mihi's own v1.0 API freeze.
- **ai-hwaccel 2.2.6** — pulled in transitively (mihi's
  `dist/mihi.cyr` references ai-hwaccel symbols in its GPU block).
  Matches mihi 1.0.0's own transitive pin; the no-exec API is the
  only ai-hwaccel surface mihi reaches.

## Consumers

_None yet._ iam is end-user-facing; "consumers" are user MOTD
invocations and shell login scripts.

## Benchmarks

Four-point cold-start trend on archaemenid (median of three N=500
batches per build, full methodology in
[`../benchmarks.md`](../benchmarks.md)):

| Build              | Per-invocation | Lines |
| ------------------ | --------------:| -----:|
| v0.3.0 (M1+M2)     |      ~555 µs   |     6 |
| M3 @ 9df0859       |     ~1514 µs   |     7 |
| v0.5.0 (M4)        |     ~1511 µs   |     7 |
| v0.9.0 (M6 RC)     |     ~1510 µs   |     7 |
| v1.0.0 (M6)        |     ~1510 µs   |     7 |

M5 < 10 ms cold-start gate met with ~6.6× headroom at v1.0.0. GPU
probe (M3) remains the dominant cost; single-flush refactor (M4)
was nominal at this scale; F-001 sanitizer (v0.9.0) is invisible
at this scale (per-byte filter cost < bench resolution). v1.0.0
inherits the v0.9.0 measurement — `src/` is byte-identical and
the mihi 1.0.0 bundle is module-content byte-identical to 0.7.0
(only the `# Version:` header stamp differs in `dist/mihi.cyr`),
so the iam binary's instruction stream is unchanged.

## Audit

v1.0.0 mihi-major-bump audit at
[`../audit/2026-05-20-v1.0.0-audit.md`](../audit/2026-05-20-v1.0.0-audit.md)
supersedes the v0.9.0 doc
([`../audit/2026-05-19-v0.9.0-audit.md`](../audit/2026-05-19-v0.9.0-audit.md))
— which superseded the M5.5 doc
([`../audit/2026-05-19-m5.5-audit.md`](../audit/2026-05-19-m5.5-audit.md))
— which superseded the M5 doc
([`../audit/2026-05-19-audit.md`](../audit/2026-05-19-audit.md))
— for the v1.0.0 scope. All four prior/current docs stay in the
directory as historical record per audit-trail convention.
**Verdict**: pass, no new findings; mihi 0.7.0 → 1.0.0 is a clean
repin.

- **F-001** (**RESOLVED at v0.9.0**, status unchanged at v1.0.0):
  TTY-escape sanitization mitigation in `iam_copy_value`
  (`src/display.cyr`); 15 byte-exact tests lock the behavior.
- **F-002** (INFO, carries forward): `strlen` on mihi cstrings is
  trust-dependent — mihi-side invariant, not iam's to fix. At
  v1.0.0 the cstring contract is **formally version-pinned** by
  mihi's own v1.0 API freeze (signature / return-shape / error-
  semantics changes now require a major mihi bump). F-001's
  sanitizer continues to provide incidental defense-in-depth
  against embedded NUL bytes surviving past `strlen`.
- **Source re-walk** (v1.0.0): zero `src/*.cyr` changes since the
  v0.9.0 RC baseline. The v1.0.0 working-tree delta is one line in
  `cyrius.cyml` (`[deps.mihi] tag` 0.7.0 → 1.0.0) plus the
  deps-managed `lib/mihi.cyr` regen. mihi-call surface, buffer
  caps, and syscall family all byte-identical to v0.9.0.
- **External CVE pass** (v1.0.0): clean. mihi 1.0.0's bundle
  carries forward the 0.6.0 parser-overflow hardenings (C-1, M-1,
  C-2) — reduces attack surface against adversarial `/proc`
  content. The two AMD GPU kernel CVEs (CVE-2025-40289 /
  CVE-2025-40288) remain environmental; archaemenid 7.0.5
  unaffected.
- **Bench impact**: median 1510 µs at v1.0.0 — inherited from
  v0.9.0 (binary byte-identical). F-001 sanitizer still invisible
  at this scale.

All other categories clean (bounds, exit-code discipline, no
unsafe syscalls, no env / file / network I/O).

**v1.0.0 release-gate status**: **clear.** Output-shape contract
(ADR 0002) is frozen from this release — any future change to the
line spine, label format, `(unknown)` fallback, or exit-code
discipline is a real `Breaking` requiring a major-version bump.

## Next

See [`roadmap.md`](roadmap.md). v1.0.0 closes M6 and shifts iam
into **post-v1.0 stewardship mode**:

- **Output-shape contract is frozen.** Any future change to line
  order, label width, label spelling, `(unknown)` fallback, or
  exit-code discipline is a real `Breaking` requiring a major-
  version bump (v2.0+). The 39 byte-exact ADR-contract assertions
  in `tests/iam.tcyr` continue to lock the shape mechanically.
- **Re-audit triggers**: mihi major bump, ai-hwaccel or cyrius
  major bump, or any source change in `src/*.cyr` beyond
  docstring / comment edits. Audit template per
  [`../audit/2026-05-20-v1.0.0-audit.md`](../audit/2026-05-20-v1.0.0-audit.md).
- **Deferred-feature reopening**: the roadmap's "Deferred" entries
  (network probes, runtime state — Battery / Swap / Disk) stay
  reopenable. Any addition that disturbs the v1.0 spine is a
  major-version bump from here; an addition that fits cleanly
  alongside the spine (e.g. a new line that doesn't reorder
  existing ones) needs ADR justification and re-audit but can
  ship in a `Minor` cut.
- **Dogfood** stays live — `~/.local/bin/iam` → `build/iam`,
  fired from `~/.zshrc` on every interactive shell. Future cuts
  auto-propagate via the symlink.
