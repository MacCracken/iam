# 0002 — Output shape (reorder for fastfetch-similar top-down layout)

**Status**: Accepted (2026-05-19)
**Date**: 2026-05-19
**Supersedes**: [0001](0001-output-shape.md) §2-§3 (line order and optional-line policy). ADR 0001 §1, §4, §5, §6 carry over unchanged.

## Context

[ADR 0001](0001-output-shape.md) locked the v1.0 byte contract:
six required labels plus an optional trailing `GPU:`, in the order
**CPU / Memory / Kernel / Host / Distro / Uptime / GPU**. That order
codified what the M1+M2 (v0.3.0) and M3 (v0.4.0) cuts had already
shipped — it captured *what existed*, not *what was easiest to read
top-to-bottom*.

Comparing iam's output against fastfetch's revealed the issue:

```
fastfetch (macOS):           iam (current):
  OS                           CPU
  Host                         Memory
  Kernel                       Kernel
  Uptime                       Host
  Packages (config)            Distro
  Shell    (config)            Uptime
  Display  (config)            GPU (optional, trailing)
  ...
  CPU      (hardware)
  GPU      (hardware)
  Memory   (hardware)
  ...
```

fastfetch has a clear zoom-in gradient: **identity** (OS, Host) →
**runtime** (Kernel, Uptime) → config block → **hardware** (CPU,
GPU, Memory) → state/network. iam's current order zigzags by
contrast — hardware first, then kernel, back to identity, then
state, then hardware again (GPU is split from CPU by four lines).
The split is also why the optional-line rule has its weird "always
position 7" carve-out: GPU couldn't live next to CPU without
breaking position-anchored parsers, so it got exiled to the end.

**Design goal: similar, not identical.** iam strips out everything
between fastfetch's Uptime and CPU lines (Packages, Shell, Display,
WM, Theme, Font, Cursor, Terminal — all configuration, on the
[roadmap "Not iam's job"](../development/roadmap.md#not-iams-job-use-the-right-tool)
list) and most of what trails Memory (Swap, Disk, Local IP,
Battery, Locale — a mix of runtime state and config; the runtime-
state pieces are
[deferred](../development/roadmap.md#deferred-may-reopen-later)
post-v1.0, network is deferred until kernel bring-up). What's left
is fastfetch's identity → runtime → hardware spine, with iam's
six required lines plus GPU sitting in their fastfetch-
corresponding slots. A user coming from fastfetch sees the same
overall layout and knows where to look without needing to learn
iam's own ordering.

This ADR proposes a single reorder before the M6 v1.0 freeze. After
v1.0, the byte contract is locked and a change like this would
require a `Breaking` CHANGELOG entry. Doing it now — while consumer
count is _zero_ per [state.md](../development/state.md) — costs
nothing externally and produces a cleaner contract to freeze.

The bytes-as-contract framing from ADR 0001 §1, §4, §5, §6 carries
over unchanged. Only §2 (line order) and §3 (optional-line policy)
are reopened.

## Decision

iam emits exactly the following bytes to stdout, in order, on every
successful run:

```
Distro: <distro_value>\n
Host:   <host_value>\n
Kernel: <kernel_value>\n
Uptime: <uptime_value>\n
CPU:    <cpu_value>\n
GPU:    <gpu_value>\n        (iff mihi_gpu_count() > 0)
Memory: <memory_value>\n
```

The decisions that carry over unchanged from ADR 0001 (8-byte label
column, `(unknown)` fallback, no color / TTY-equivalence, exit-0
contract) are repeated here for completeness so 0002 is the single
source consumers need to read.

### 1. Label column (unchanged from 0001 §1)

Each line starts with a label followed by `:`, then ASCII spaces
padded to a total of **8 bytes** before the value. Padding is spaces,
never tabs. The padding holds for the unchanged label set:

| Label    | `:` + pad | Column |
|----------|-----------|--------|
| `Distro` | `: `      | 8      |
| `Host`   | `:   `    | 8      |
| `Kernel` | `: `      | 8      |
| `Uptime` | `: `      | 8      |
| `CPU`    | `:    `   | 8      |
| `GPU`    | `:    `   | 8      |
| `Memory` | `: `      | 8      |

### 2. Line order (changed from 0001 §2)

The required six labels appear in this exact order:

1. `Distro` — identity (≈ fastfetch's `OS` slot)
2. `Host`   — identity (≈ fastfetch's `Host` slot)
3. `Kernel` — what's running (≈ fastfetch's `Kernel` slot)
4. `Uptime` — runtime state (≈ fastfetch's `Uptime` slot)
5. `CPU`    — hardware (≈ fastfetch's `CPU` slot, first hw line)
6. `Memory` — hardware (≈ fastfetch's `Memory` slot)

Rationale: matches the **identity → runtime → hardware** spine of
fastfetch's familiar top-down ordering. A reader scanning iam sees
the same gradient they're used to from fastfetch, in the same
relative slots — Distro where they expect OS, Uptime where they
expect Uptime, hardware block trailing. iam differs by *omitting*
fastfetch's config and network blocks entirely (Packages, Shell,
Display, WM, Theme, Font, Cursor, Terminal, Swap, Disk, Local IP,
Battery, Locale — all out of scope per roadmap), not by reshuffling
what's left.

The optional `GPU` line, when present, slots between `CPU` and
`Memory` — i.e. **position 6** when GPU is detected, otherwise
absent and `Memory` slides up to position 6. This puts GPU in the
fastfetch-corresponding slot (between CPU and Memory) and keeps
the hardware block contiguous. See §3.

### 3. Optional-line policy (changed from 0001 §3)

`GPU:` remains the **only** optional line in iam. It is emitted iff
`mihi_gpu_count()` returns a value greater than zero. On a host
without a detectable accelerator the line is absent and the output
is exactly six lines.

The change vs ADR 0001: GPU now appears *between CPU and Memory*
instead of at the trailing position. Rationale:

- **fastfetch-similar slot**: fastfetch emits GPU between CPU and
  Memory. iam follows the same hardware-block ordering so a
  fastfetch user reading iam sees GPU where they'd expect it.
- **Semantic adjacency**: GPU is hardware. CPU and Memory are
  hardware. Grouping them puts the hardware block in one place
  rather than splitting it.
- **No special-case in the renderer**: ADR 0001's "always position
  7" rule meant the driver had to render six lines, then maybe
  append a seventh. The new rule is "after CPU, conditionally
  emit GPU, then continue" — the same insertion shape used for
  any conditional line, no longer a tail-append carve-out.
- **Parser cost**: low. Consumers that anchored on "GPU is line
  7 if present" must switch to "GPU is line 6 if present". The
  six-or-seven-line count guarantee is unchanged; label-based
  matching (`grep ^GPU:`) is unchanged.

For multi-GPU hosts, iam still emits one `GPU:` line containing
the first detected device (`mihi_gpu_name(0)`) — unchanged from
ADR 0001 §3.

### 4. Unknown-value fallback (unchanged from 0001 §4)

When a probe returns null (`0`) or a negative sentinel, iam emits
the literal byte string `(unknown)` in the value column. The line
itself always appears (with the exception of the optional GPU rule
above) — the contract is "six labels always, in order"; suppressing
a required line on probe failure would break the byte-position
guarantee that consumers rely on.

`(unknown)` is the only placeholder string. iam does not distinguish
between "probe returned null" and "probe ran but found nothing" —
mihi is the layer that owns probe semantics.

### 5. No color, no escape sequences, no TTY detection (unchanged from 0001 §5)

The output is plain ASCII / UTF-8 byte text with newline (0x0A)
terminators. iam emits identical bytes on a terminal and through a
pipe — `iam` and `iam | cat` produce byte-for-byte equal stdout.

Color support is explicitly out of scope for v1.0 per the roadmap.
If reconsidered for v2.0, color additions would require a new ADR
and would default to off (preserving the v1.0 byte contract).

### 6. Exit code (unchanged from 0001 §6)

iam exits 0 on every successful run. Probe failure is signaled in
the output (`(unknown)`), not in the exit code — login MOTD scripts
running under `set -e` must not be tripped by a transient
`/proc/meminfo` read error. iam reserves non-zero exit codes for
catastrophic failure (out-of-memory during probe, mihi linkage
failure); no code path in iam returns a non-zero value to satisfy
a documented condition.

## Consequences

- **Positive** — fastfetch-similar layout. A user coming from
  fastfetch reads iam without re-learning ordering: Distro/Host
  at top, Kernel/Uptime next, hardware block at the bottom. iam
  earns familiarity by adopting the parts of fastfetch's spine
  that fit whoami-simple, while still being its own (much smaller)
  tool.

- **Positive** — top-down reading gradient. Identity → runtime →
  hardware. A user scanning iam output sees the answer to "what
  is this box" before the answer to "what CPU does it have" —
  the natural reading direction for a whoami-style identifier
  tool.

- **Positive** — hardware block is contiguous. CPU / GPU / Memory
  sit together. Anyone scanning for "what's the silicon" reads one
  cluster instead of three scattered lines.

- **Positive** — renderer simplification. The "always position 7"
  GPU rule in the driver (main.cyr:110–114, a trailing conditional
  append) becomes an inline conditional between CPU and Memory.
  Same lines of code, but the optional-line policy is no longer a
  special end-of-output case.

- **Negative** — invalidates byte-exact tests. The 39 ADR-contract
  assertions in `tests/iam.tcyr` (six-line and seven-line assembled
  output) must be regenerated against the new order. Per-label
  padding tests are unaffected (label set didn't change).

- **Negative** — invalidates the canonical sample at
  `docs/examples/sample-output.txt`. Must be regenerated.

- **Negative** — breaks position-anchored consumers. Per
  [state.md](../development/state.md), iam currently has zero
  consumers; this cost is hypothetical at the moment of the
  change. After v1.0 the same change would be expensive.

- **Neutral** — CHANGELOG gets a `Breaking` entry (still pre-v1.0,
  so the rule is informal — but worth marking explicitly so the
  pre-v1.0 → v1.0 reader sees that the contract moved once
  deliberately before the freeze).

- **Neutral** — the M5 audit (`docs/audit/2026-05-19-audit.md`)
  references main.cyr line numbers that will shift slightly when
  the driver is rewritten. The audit's findings (F-001, F-002)
  are unaffected; only line numbers need refresh.

## Migration

If accepted, the implementation bite is:

1. **Update `src/main.cyr`** — reorder the `iam_render*` calls in
   the seven-line accumulation block to match the new sequence.
   The `mihi_*` probe calls don't need to move; only the order of
   `iam_render(&out + pos, …)` calls changes.
2. **Update `tests/iam.tcyr`** — regenerate the six-line and
   seven-line assembled-output assertions (~iam.tcyr:179–224)
   against the new order. Per-label padding tests are unchanged.
3. **Update `docs/examples/sample-output.txt`** — regenerate by
   running `./build/iam > docs/examples/sample-output.txt`.
4. **Update `docs/development/state.md`** — `## Output` section's
   sample matches the new order.
5. **Update `docs/adr/0001-output-shape.md`** — change `Status`
   from `Accepted` to `Superseded by 0002`. Body left intact as
   the historical record.
6. **Update `docs/adr/README.md`** — index 0002 alongside 0001.
7. **CHANGELOG `[Unreleased]`** — `### Breaking` section noting
   the reorder; reference 0002.

Sized as a small bite (one driver edit, one test regenerate, three
doc edits). Cleanly contained in a single cut, candidate for v0.6.0.

## Alternatives considered

- **Keep ADR 0001 order**. Rejected — the zigzag (hardware → kernel
  → identity → state → hardware) costs every reader a small mental
  reordering on every invocation. iam is on the login hot-path; a
  one-time fix avoids paying that cost in perpetuity. The cost of
  the reorder is *also* one-time but bounded to zero current
  consumers — strictly cheaper to do now than at v2.0.

- **Move OS to top but keep GPU trailing**. Rejected — the half-fix.
  The top-down reading order argument applies just as much to "GPU
  next to CPU" as it does to "Distro at top". Doing one without the
  other locks in a contract that's better than 0001 but worse than
  it could be, with the same cost (an ADR + a test regen + a
  CHANGELOG breaking entry).

- **Add a `Boot` or `Init` line near `Uptime`**. Rejected — feature
  creep. CLAUDE.md's *feature-creep gate* says: "could this go in
  a sibling tool instead?" Yes — a boot-timing tool isn't iam.

- **fastfetch's exact ordering** (OS, Host, Kernel, Uptime,
  Packages, Shell, Display, …, CPU, GPU, Memory, …). Rejected
  for the middle and tail — Packages / Shell / Display and
  friends are config (roadmap's "Not iam's job"), the
  runtime/state items are deferred to post-v1.0, network is
  deferred until kernel bring-up. The *ordering of what
  remains* is what this ADR adopts: iam's six required lines
  plus optional GPU sit in their fastfetch-corresponding slots
  (identity → kernel → uptime → hardware), giving fastfetch
  users muscle-memory familiarity without iam having to grow
  fastfetch's surface area.

- **Uptime trailing** (Distro / Host / Kernel / CPU / GPU /
  Memory / Uptime, as an earlier draft of this ADR proposed).
  Rejected — diverges from fastfetch's Uptime-after-Kernel slot,
  forfeits the "similar layout" benefit. Putting Uptime with the
  identity/runtime block also reads more naturally: "what is
  this, what's running, how long, what hardware" beats "what
  is this, what's running, what hardware, oh and uptime."

- **Two-column layout** ("Distro / Host" on one line). Rejected —
  breaks the line-equals-one-fact contract. Parsers that
  `grep ^Distro:` would no longer work. Whoami-simple is one
  fact per line.

## References

- [ADR 0001](0001-output-shape.md) — predecessor; this ADR
  supersedes §2 and §3 only.
- `docs/development/state.md` — current output sample (will
  need regen).
- `docs/examples/sample-output.txt` — canonical sample (will
  need regen).
- `docs/benchmarks.md` — perf characteristics; unaffected (line
  count and probe set are unchanged).
- `docs/audit/2026-05-19-audit.md` — security audit; findings
  unaffected, line numbers will shift slightly.
- `CLAUDE.md` — load-bearing principles; this ADR is downstream
  of the "keep it whoami-simple" rule.
