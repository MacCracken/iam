# iam — Current State

> Refreshed every release. CLAUDE.md is preferences/process/procedures
> (durable); this file is **state** (volatile).

## Version

**Current release**: **1.1.7** — shipped 2026-08-23. **P(-1)
hardening sweep** — five findings, one a live contract violation, all
fixed and regression-tested. Full write-up in
[`../audit/2026-08-23-v1.1.7-audit.md`](../audit/2026-08-23-v1.1.7-audit.md).
**F-003 (HIGH)**: nothing bounded a probe value against the 4 KiB
output buffer, so an over-long value made `iam_render` return its error
sentinel, the driver's `if (n > 0)` left the cursor unmoved, and a
*required* line vanished — five lines instead of six, silently, against
an ADR 0002 guarantee. Fixed with `IAM_VALUE_MAX = 256` enforced in
`iam_copy_value` (the one choke point all four renderers share), with
UTF-8-safe truncation. **F-004 (MEDIUM)**: the report was flushed with
a single unchecked `write(2)` — `print()` discards the return, twice
down — so a short write truncated it and `-EINTR` lost it; `iam_flush`
now loops, retries `-EINTR`, and is verified not to hang on EPIPE.
**F-005/F-006/F-007 (LOW)**: DEL (0x7F) missed by the C0 sanitizer,
dead `iam_append`, and copy primitives that accepted a negative length
and silently rewound the write cursor. Tests **122 → 141**, every one
confirmed red against the v1.1.6 source first. `src/main.cyr` and
`tests/iam.tcyr` normalized to canonical cyrfmt layout, closing the
last cleanliness gap. Runtime output byte-identical to v1.1.6;
benchmark 1580 → 1573 µs (noise).

**Previous**: 1.1.6 — shipped 2026-08-23. Two cuts in one.
(1) The `GPU:` line gained an on-device-memory suffix
(`AMD Radeon (PCI 0x1002:0x1638) [3 GiB]`) per
[`docs/adr/0003-gpu-line-memory-suffix.md`](../adr/0003-gpu-line-memory-suffix.md);
iam had consumed `mihi_gpu_count()` / `mihi_gpu_name()` since v0.4.0
and ignored `mihi_gpu_memory_bytes()`, and wiring it in is also what
carries the AGNOS kernel's `gpu_caps` **`vram_mb`** field (syscall
**#89**, `len >= 96` identity tier) up to a printed line. New
`iam_render_gpu` renderer in `src/display.cyr`; tests 105 → **122**.
Degrades silently to the bare device name where the probe carries no
size, so hosts that learned nothing emit v1.1.5's exact bytes.
(2) **Toolchain + dep floor refresh**: `[package].cyrius` 6.2.37 →
**6.5.35** (closes the wrapper/manifest drift), `[deps.mihi]` 1.2.1 →
**1.2.4**, `[deps.ai-hwaccel]` 2.2.6 → **2.3.18** (matches mihi
1.2.4's own transitive), `sakshi` added to the stdlib list — which is
now byte-equal to mihi's `dist/mihi.deps` sidecar, 21 modules.
`cyrius lib sync --full` re-vendored the 6.5.35 snapshot (108 files,
adds `lib/unicode/` + `async_macos` / `async_win` / `thread_macos`)
and ten orphaned pre-6.2.x modules were pruned — which nets out to
an unchanged **110**-entry lock, ten files out and ten in.
mihi's probe API is unchanged, so **zero `src/*.cyr` changes in the
refresh half** and rendered output is byte-identical on archaemenid.
(3) **CI now installs the toolchain with cyrius's own installer**
rather than a hand-rolled `curl`+`tar`+`cp`, which is what makes the
new vendored-`lib/` drift gate possible at all — see *Toolchain*.
Line order, label set, label width, `(unknown)` policy, and exit-0
are untouched — output is still six or seven lines. `Minor`-eligible
per the stewardship clause below.

**Previous**: 1.1.5 — shipped 2026-07-02. Fixed the agnos
user-stack overflow that made `run /bin/iam` fault before printing
anything (heap-allocated the 8 KiB cpu + 4 KiB mem `/proc` scratch
buffers, dropping the `main` frame from ~17.6 KB to ~5.5 KB against
agnos's ~12 KB budget) and bumped `[deps.mihi]` to **1.2.1** for the
CPUID brand-string fix that actually compiles under `--agnos`. The
1.1.2 → 1.1.4 cuts were mihi repins (agnos build-target probes,
sovereign CPUID CPU-model probe) and the `agnosys` → `sys` stdlib
rewire. Full history in
[`../../CHANGELOG.md`](../../CHANGELOG.md); the per-release detail
below stops at 1.1.1 and has not been backfilled.

**Previous**: 1.1.1 — shipped 2026-06-18. **Toolchain + dep refresh.**
`[package].cyrius` 6.0.1 → **6.2.22** (matches mihi 1.1.1's pin),
`[deps.mihi]` 1.0.0 → **1.1.1**. The 6.2.x stdlib reorg lands:
`agnosys` leaves the stdlib and becomes a git dep
(`[deps.agnosys] 1.4.0`, `dist/agnosys-core.cyr`) because mihi
1.1.1 reads `uname`/`sysinfo` through `agnosys_uname`; `json` →
`bayan` in the stdlib list (json folded into the bayan bundle via
sandhi, back-compat aliases resolve mihi's `registry_to_json`).
`[deps.ai-hwaccel]` stays **2.2.6** — mihi 1.1.1 pins the same
transitive (the 2.3.x line is not referenced). `cyrius lib sync`
re-vendored the 6.2.22 snapshot into `lib/`; `cyrius.lock`
regenerated (110 deps). **Zero `src/*.cyr` changes** — runtime
output byte-for-byte identical to v1.1.0 on archaemenid; build,
lint, tests pass.

**Previous**: 1.1.0 — 2026-06-06. Cycle-open for AGNOS as a build
target (VERSION → 1.1.0): an AGNOS-target build so `iam` renders
natively on AGNOS off the `uname`#34 / `sysinfo`#35 kernel
syscalls; GPU/distro lines suppress where AGNOS has no source.
Inline, no platform-abstraction layer.

**Previous**: 1.0.0 — shipped 2026-05-20. **M6 closed. Output-shape
contract frozen.** Lockstep release with mihi 1.0.0. The cut is the
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

- **Cyrius pin**: `6.5.35` (in `cyrius.cyml [package].cyrius`) — the
  single source of truth. No workflow YAML hardcodes a version; both
  `ci.yml` and `release.yml` read the pin out of the manifest.
- **Install path (CI and local)**: cyrius's own
  `scripts/install.sh`, invoked with `CYRIUS_VERSION` set to the pin.
  This is load-bearing, not cosmetic — the installer lays out
  `~/.cyrius/versions/<v>/{bin,lib}`, and that **versioned snapshot**
  is what `cyrius lib sync` reads from and what the compiler diffs
  `./lib/` against to emit `./lib/ shadows version-pinned ...`. A
  hand-rolled `curl` + `tar` + `cp` populates only `~/.cyrius/{bin,lib}`,
  which leaves `cyrius lib sync` failing outright (`snapshot lib not
  found at ~/.cyrius/versions/<v>/lib`) and the shadow check with
  nothing to compare against. CI carried exactly that hand-rolled step
  until 1.1.6, which is why the vendored-`lib/` drift found at that cut
  was invisible to CI by construction.
- **Vendored `lib/` is gated.** CI runs `cyrius lib sync --full` and
  fails if `git status --porcelain lib/` is non-empty — re-syncing a
  correct tree must be a no-op. Reproduce locally with the same two
  commands before committing a pin bump.

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
  buffer, flushes once via `iam_flush` (short-write- and
  `-EINTR`-safe; v1.1.7 F-004). Optional GPU line is inlined here
  (single probe, single line, suppress-on-zero). `IAM_EINTR` is
  defined locally because the stdlib's `EINTR` does not exist on the
  agnos target.
- `src/display.cyr` — `iam_render` / `iam_render_buf` /
  `iam_render_kernel` / `iam_render_gpu` line renderers
  (write-into-buffer, return bytes-written), `iam_format_bytes`
  (binary units, floor), `iam_uint_into` shared digit helper,
  `iam_put` / `iam_copy` internal append+bounds primitives.
  `iam_render_gpu` composes device name + optional bracketed size
  (ADR 0003) and degrades to the bare name when the size is absent.
  `iam_copy_value` is the single choke point every value column
  routes through: it sanitizes C0 + DEL and enforces
  `IAM_VALUE_MAX` (256 bytes, UTF-8-boundary-safe) so an over-long
  probe value truncates instead of costing a whole required line
  (v1.1.7 F-003).
- `src/uptime.cyr` — `iam_format_uptime` (seconds → "1d 2h 3m" with
  zero-elision + `<1m` floor).

## Output

Current sample (on archaemenid, 2026-08-01):

```
Distro: Arch Linux
Host:   archaemenid
Kernel: Linux 7.1.5-arch1-1
Uptime: 20m
CPU:    AMD Ryzen 7 5800H with Radeon Graphics
GPU:    AMD Radeon (PCI 0x1002:0x1638) [3 GiB]
Memory: 59 GiB
```

On a GPU-less host (most servers, hosted CI runners) the GPU line
is suppressed and `Memory` slides up to position 6; the output is
six lines instead of seven. Where an accelerator is present but its
on-device memory is unknown, the `[3 GiB]` suffix is omitted and the
line carries the bare device name (ADR 0003) — byte-identical to
what iam emitted before the suffix existed.

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
  block contiguous. Its value column is the first device's name
  plus, when the probe knows it, a bracketed on-device-memory
  suffix (ADR 0003). Multi-GPU hosts still get exactly one GPU
  line — the six-or-seven-line count guarantee is part of the
  frozen shape.
- Exit 0 always — probe failure is signaled in `(unknown)`, not
  the exit code (login MOTD under `set -e` must not be tripped).

## Tests

- `tests/iam.tcyr` — 141 assertions:
  - Formatter logic (51): `iam_uint_into`, `iam_format_bytes`
    (boundary + floor + preview cases), `iam_format_uptime`
    (zero-elision, fresh-boot floor, cap-too-small).
  - ADR-contract byte-exact (41): per-label padding for all seven
    labels, `(unknown)` fallback paths through every renderer
    variant, full six-line + two seven-line assembled outputs (GPU
    size present / absent), full all-probes-failed degraded shape.
    Regenerated at v0.8.0 against the ADR 0002 order.
  - F-001 TTY-escape sanitization byte-exact (15): `iam_copy_value`
    boundary (0x1F / 0x20 / TAB); ESC in `iam_render` value;
    newline-injection attempt; ESC mid-buffer in `iam_render_buf`;
    ESC + TAB in `iam_render_kernel` name + version; unchanged
    `IAM_UNKNOWN_TEXT` fallback path.
  - ADR 0003 GPU value column (15): bracketed size suffix, the real
    archaemenid parenthesised-PCI-id device name, both size-absent
    degrade paths (`mlen = 0` and mihi's `0 - 1` sentinel),
    `name = 0` with and without a size, overflow signaling, ESC
    sanitization on the composed line.
  - v1.1.7 P(-1) hardening (19): the six-line contract surviving a
    pathological 5000-byte value, `IAM_VALUE_MAX` holding across all
    four renderers, a value exactly at the cap (off-by-one), UTF-8
    boundary back-off on truncation, DEL (0x7F) sanitization, UTF-8
    bytes surviving the sanitizer, and copy-primitive bounds at a
    non-zero cursor. Each was confirmed to **fail** against the v1.1.6
    source before being accepted.
- `tests/iam.bcyr` — benchmark stub (`noop` micro-bench).
- `tests/iam.fcyr` — fuzz stub.

End-to-end probe wiring is verified by building the binary and
running it on the maintainer's box (archaemenid). The byte-exact
suite covers the ADR contract executable-form for synthetic inputs.

## Dependencies

Direct (declared in `cyrius.cyml`):

- **stdlib (21 modules)** — string, fmt, alloc, io, vec, str, slice,
  syscalls, sys, assert, fs, tagged, process, fnptr, thread,
  freelist, hashmap, sakshi, ct, bayan, bench. This list is the
  union of iam's own needs and what the mihi + ai-hwaccel bundles
  reference, and as of 1.1.6 it is byte-equal to mihi's
  `dist/mihi.deps` sidecar (auto-generated by `cyrius distlib`,
  consumed by `cyrius deps`) — so it is checkable rather than
  guessed. Notes on the non-obvious entries: `sys` carries the
  native `uname`/`sysinfo` plumbing mihi reads through `sys_uname` /
  `sys_sysinfo` (it is opt-in-vendored — needs `cyrius lib sync`);
  `json` → `bayan` since the 6.2.x reorg folded json/toml/cyml/csv/
  base64/bigint/u128 into the bayan distribution; `sakshi` joined at
  1.1.6 because ai-hwaccel 2.3.x routes detect-path diagnostics
  through it.
- **mihi 1.2.4** — the probe library; every displayed line is a
  `mihi_*` call. Contract-bound per mihi's v1.0 API freeze — the
  eleven symbols iam calls (`mihi_uname`, `mihi_hostname`,
  `mihi_kernel_name`, `mihi_kernel_version`, `mihi_distro`,
  `mihi_uptime_secs`, `mihi_cpu_model`, `mihi_mem_total`,
  `mihi_gpu_count`, `mihi_gpu_name`, `mihi_gpu_memory_bytes`) have
  been signature-stable since 1.0.0. 1.2.4 adds an aarch64
  device-tree source for `mihi_cpu_model`, so an arm64 Linux box
  booted via device tree now renders a real `CPU:` value instead of
  `(unknown)`; ACPI-booted arm64 still has no source.
- **ai-hwaccel 2.3.18** — pulled in transitively (mihi's
  `dist/mihi.cyr` references ai-hwaccel symbols in its GPU block).
  Matches mihi 1.2.4's own transitive pin. iam references no
  ai-hwaccel symbol directly; the bundle is present only so mihi's
  `gpu.cyr` parses, and DCE drops the unreached code from the
  binary. The no-exec API is the only surface mihi reaches.

`agnosys` is **not** a dependency and has not been since 1.1.3, when
mihi rewired to `sys_uname` / `sys_sysinfo`. The last physical trace
(the orphaned `lib/agnosys-core.cyr`) was pruned at 1.1.6.

Vendored (`lib/`, managed by `cyrius deps` / `cyrius lib sync`):
110 files — the 108-module 6.5.35 stdlib snapshot (including the
`lib/unicode/` sub-package) plus `mihi.cyr` and `ai-hwaccel.cyr`.
`lib/` matches the pinned snapshot exactly as of 1.1.6; a mismatch
makes `cyrius build` emit a `./lib/ shadows version-pinned` warning,
which is the signal to re-run `cyrius lib sync --full`.

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
| v1.1.6 (refresh)   |     ~1578 µs   |     7 |

M5 < 10 ms cold-start gate met with ~6.6× headroom at v1.0.0. GPU
probe (M3) remains the dominant cost; single-flush refactor (M4)
was nominal at this scale; F-001 sanitizer (v0.9.0) is invisible
at this scale (per-byte filter cost < bench resolution). v1.0.0
inherits the v0.9.0 measurement — `src/` is byte-identical and
the mihi 1.0.0 bundle is module-content byte-identical to 0.7.0
(only the `# Version:` header stamp differs in `dist/mihi.cyr`),
so the iam binary's instruction stream is unchanged.

v1.1.6's row is **not** comparable to the four above it — different
kernel (7.1.8 vs 7.0.5) and different toolchain (6.5.35 vs 6.0.1).
The question that cut actually had to answer was whether mihi 1.2.3's
deliberate 126% `mihi_cpu_model` slowdown (it had been reading 3288 of
the 8192 bytes asked for; now it reads all of them) costs iam anything,
and that was measured as an interleaved A/B with cycc held constant at
6.5.35: **1628 µs** median on the 1.1.5 pins vs **1578 µs** on the
1.1.6 pins, three N=500 trials each. New is marginally faster, which
is noise — the finding is that the cpu_model change is invisible at
iam's scale because the GPU probe still dominates. M5's < 10 ms gate
holds with ~6.3× headroom.

## Audit

**Current**: v1.1.7 P(-1) hardening audit at
[`../audit/2026-08-23-v1.1.7-audit.md`](../audit/2026-08-23-v1.1.7-audit.md),
superseding the v1.0.0 doc
([`../audit/2026-05-20-v1.0.0-audit.md`](../audit/2026-05-20-v1.0.0-audit.md))
— which superseded v0.9.0
([`../audit/2026-05-19-v0.9.0-audit.md`](../audit/2026-05-19-v0.9.0-audit.md)),
M5.5
([`../audit/2026-05-19-m5.5-audit.md`](../audit/2026-05-19-m5.5-audit.md))
and M5
([`../audit/2026-05-19-audit.md`](../audit/2026-05-19-audit.md)).
All five stay in the directory as historical record per audit-trail
convention.

**Verdict: not a pass on arrival** — the first audit since v1.0.0 to
re-walk `src/` line-by-line found five issues, one of them a live
violation of the frozen output contract. All five are fixed and
regression-tested in the same cut; post-fix the tree is clean on every
gate.

- **F-003** (**HIGH, fixed at v1.1.7**): an over-long probe value
  overflowed the 4 KiB output buffer, and because the driver only
  advances its cursor on a positive renderer return, the whole
  *required* line was dropped — five lines instead of six, silently,
  against ADR 0002's guarantee. Bounded at display time by
  `IAM_VALUE_MAX = 256` in `iam_copy_value`, UTF-8-boundary-safe.
  Reproduced with a 5000-byte CPU model before the fix.
- **F-004** (MEDIUM, fixed at v1.1.7): the report was flushed with one
  unchecked `write(2)` — short write truncates, `-EINTR` loses it.
  `iam_flush` loops and retries; verified not to hang on EPIPE or a
  closed stdout.
- **F-005 / F-006 / F-007** (LOW, fixed at v1.1.7): DEL (0x7F) missed
  by the C0 sanitizer; `iam_append` dead code; copy primitives that
  accepted a negative length and silently rewound the write cursor.
- **F-001** (RESOLVED at v0.9.0): TTY-escape sanitization in
  `iam_copy_value`. **Extended at v1.1.7** to cover DEL. C1
  (0x80–0x9F) is deliberately *not* filtered — on a UTF-8 terminal
  those are continuation bytes, so filtering them would corrupt
  legitimate non-ASCII values while protecting against nothing.
- **F-002** (INFO, carries forward): `strlen` on mihi cstrings is
  trust-dependent — a mihi-side invariant, not iam's to fix, and
  formally version-pinned by mihi's v1.0 API freeze. F-003's clamp
  bounds how much of an over-long value is *copied* but does **not**
  bound the `strlen` scan itself; closing F-002 needs an `n`-limited
  probe variant from mihi.
- **Why four prior audits missed F-003**: F-002's framing ("iam trusts
  mihi's NUL invariant") anchored every subsequent review on
  *malformed* input. F-003 needs nothing malformed — a correctly
  NUL-terminated 5000-byte string is enough. The unasked question was
  not "what if mihi returns something broken" but "what if mihi returns
  something **large**".
- **Syscall surface** (v1.1.7): iam's own `src/` issues exactly one
  direct syscall, `syscall(SYS_EXIT, r)`. No exec/fork/clone, no
  sockets, no `dlopen`, no `getenv`, no direct `/proc` `/sys` `/etc`
  opens, no stack buffer ≥ 64 KiB — all enforced by the CI security
  scan.
- **External research** (v1.1.7): no CVEs found against fastfetch or
  neofetch for the escape-injection class, but the class itself is
  well-attested (WinRAR CVE-2024-33899 / CVE-2024-36052 are the
  concrete precedent for exactly what F-001 mitigates). ECMA-48 was the
  source for the C0-includes-DEL correction in F-005. The dep tree is
  entirely first-party (cyrius / mihi / ai-hwaccel) with no third-party
  code, no network, and no FFI.
- **Bench impact**: 1580 µs (v1.1.6) → 1573 µs (v1.1.7), interleaved
  A/B with the compiler held constant. Hardening is free at this scale.

All other categories clean (bounds, exit-code discipline, no unsafe
syscalls, no env / file / network I/O).

**Contract status**: ADR 0002's output-shape freeze holds. F-003 was
not a contract change — it is iam starting to honour a guarantee the
ADR already made and the code could violate.

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
