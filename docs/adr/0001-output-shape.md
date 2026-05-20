# 0001 — Output shape

**Status**: Superseded by [0002](0002-output-shape-reorder.md) (2026-05-19)
**Date**: 2026-05-19

> §2 (line order) and §3 (optional-line policy) are superseded by
> ADR 0002 — output now follows the identity → runtime → hardware
> spine (Distro / Host / Kernel / Uptime / CPU / GPU? / Memory). §1
> (label column), §4 (`(unknown)` fallback), §5 (no color, TTY ==
> pipe), and §6 (exit 0) carry over unchanged and remain canonical
> here. Body left intact below as the historical record.

## Context

iam is the inverse of `whoami` — a one-shot CLI that prints what the
system is. By M4 we've shipped CPU / Memory / Kernel / Host / Distro
/ Uptime in M1+M2 (v0.3.0) and the optional GPU line in M3 (v0.4.0).
Behavior has settled in practice; consumers (MOTD scripts, login
shells) are about to start treating the bytes as a contract. This
ADR locks that contract before v1.0 freezes it.

The decisions that already exist as implicit conventions in the
code, CLAUDE.md, and CHANGELOG entries:

- 8-byte label column ("Memory" + ':' + 1 space — the worst case).
- Required label set: CPU / Memory / Kernel / Host / Distro / Uptime.
- Optional trailing GPU line when an accelerator is detected.
- `(unknown)` placeholder when a probe returns null/negative.
- No color, no escape sequences — TTY and pipe output are byte-equal.
- Exit 0 always (so MOTD scripts don't trip `set -e`).
- No flags beyond eventual `--help` / `--version`.

CLAUDE.md's load-bearing principle ("Keep it whoami-simple") and the
roadmap's "Out of scope" list (no theming, no plugins, no
configurable output, no JSON/CSV, no caching, no network) constrain
what this contract can ever look like. This ADR formalizes the
positive shape that those negatives leave.

## Decision

iam emits exactly the following bytes to stdout, in order, on every
successful run:

```
CPU:    <cpu_value>\n
Memory: <memory_value>\n
Kernel: <kernel_value>\n
Host:   <host_value>\n
Distro: <distro_value>\n
Uptime: <uptime_value>\n
GPU:    <gpu_value>\n        (iff mihi_gpu_count() > 0)
```

The contract has six load-bearing pieces:

### 1. Label column

Each line starts with a label followed by `:`, then ASCII spaces
padded to a total of **8 bytes** before the value. The padding is
spaces, never tabs. The label set is fixed (see below); the column
width must accommodate the longest label + `:` + one trailing space:

| Label    | `:` + pad | Column |
|----------|-----------|--------|
| `CPU`    | `:    `   | 8      |
| `Memory` | `: `      | 8      |
| `Kernel` | `: `      | 8      |
| `Host`   | `:   `    | 8      |
| `Distro` | `: `      | 8      |
| `Uptime` | `: `      | 8      |
| `GPU`    | `:    `   | 8      |

### 2. Line order

The required six labels appear in this exact order:

1. `CPU`
2. `Memory`
3. `Kernel`
4. `Host`
5. `Distro`
6. `Uptime`

The optional `GPU` line, when present, is always position 7. No
other line may appear between positions 1–6, and nothing trails
`GPU` (or `Uptime` when GPU is suppressed).

### 3. Optional-line policy

`GPU:` is the **only** optional line in iam. It is emitted iff
`mihi_gpu_count()` returns a value greater than zero. On a host
without a detectable accelerator the line is absent and the output
is exactly six lines.

A future line (e.g. additional GPUs, additional storage facts)
would require a new ADR and a `Breaking` CHANGELOG entry. The
single-optional-line rule keeps the shape predictable for shell
parsers that anchor on line count.

For multi-GPU hosts, iam emits one `GPU:` line containing the first
detected device (`mihi_gpu_name(0)`). Consumers needing per-device
detail call mihi directly.

### 4. Unknown-value fallback

When a probe returns null (`0`) or a negative sentinel, iam emits
the literal byte string `(unknown)` in the value column. The line
itself always appears (with the exception of the optional GPU rule
above) — the contract is "six labels always, in order"; suppressing
a required line on probe failure would break the byte-position
guarantee that consumers rely on.

`(unknown)` is the only placeholder string. iam does not distinguish
between "probe returned null" and "probe ran but found nothing" —
mihi is the layer that owns probe semantics.

### 5. No color, no escape sequences, no TTY detection

The output is plain ASCII / UTF-8 byte text with newline (0x0A)
terminators. iam emits identical bytes on a terminal and through a
pipe — `iam` and `iam | cat` produce byte-for-byte equal stdout.

Color support is explicitly out of scope for v1.0 per the roadmap.
If reconsidered for v2.0, color additions would require a new ADR
and would default to off (preserving the v1.0 byte contract).

### 6. Exit code

iam exits 0 on every successful run. Probe failure is signaled in
the output (`(unknown)`), not in the exit code — login MOTD scripts
running under `set -e` must not be tripped by a transient
`/proc/meminfo` read error. iam reserves non-zero exit codes for
catastrophic failure (out-of-memory during probe, mihi linkage
failure) — those paths today exit nonzero implicitly via the
underlying syscall failures, but no code path in iam returns a
non-zero value to satisfy a documented condition.

## Consequences

- **Positive** — consumers can parse iam output with `cut -d':' -f2-`
  / `awk -F': *' '{print $2}'` and trust the column positions.
  Login MOTD scripts can string-match on labels without worrying
  about color escapes. Snapshots in screenshots / docs are
  reproducible across machines (only the value half changes).

- **Positive** — adding a new line (or reordering) becomes a deliberate
  Breaking change requiring a superseding ADR + a CHANGELOG entry,
  not an incidental edit. The bytes-are-contract framing makes
  drift visible.

- **Negative** — every probe failure adds an `(unknown)` line that
  looks identical to a fresh box that genuinely has no data. iam is
  a presentation layer; ambiguity here is by design (mihi owns
  diagnostic surface, iam owns terseness).

- **Negative** — the 8-byte label column caps the longest label at 7
  characters (`Memory`, `Kernel`, `Distro`, `Uptime` all fit). A
  future hypothetical 8-character label (`Hardware:` ?) would force
  a column-width re-decision; this ADR is what gets superseded.

- **Neutral** — the spec creates a natural place to land
  byte-for-byte tests against a fixed mihi-mock input (roadmap M4's
  third deliverable, deferred to a follow-up bite). Those tests
  become the executable form of this document.

## Alternatives considered

- **Tab-separated label/value**. Rejected — visually noisier and
  fragile against terminal tab-width settings. Spaces give the
  reader a fixed column regardless of `tabstop`.

- **Variable label column width**. Rejected — readers can't rely on
  byte positions if a longer label slides everything right. Fixed-8
  matches the longest current label and leaves room for none longer.

- **Color on TTY, plain on pipe** (fastfetch-style). Rejected per
  CLAUDE.md "NO theming engine. NO plugin system. NO configurable
  output." Same-bytes-everywhere is the contract.

- **Always emit GPU, render `(unknown)` when absent**. Rejected —
  most servers have no accelerator and a permanent `(unknown)` line
  would be noise. Suppress-on-zero is the only optional-line policy
  iam carves out, and it's worth the small "is the GPU line there?"
  decision burden on parsers.

- **One line per GPU** on multi-accelerator hosts. Rejected — would
  break the fixed line-count contract that shell parsers rely on.
  Multi-GPU is rare enough that picking the first device is the
  whoami-simple choice; consumers needing more call mihi directly.

- **Multiple optional lines** (suppress Distro on AGNOS-native,
  suppress Uptime on stateless hosts, etc.). Rejected — every
  optional line is a fork in the byte contract. Capping at one
  optional line (GPU) keeps the consumer-side branch count at two
  (six or seven lines).

- **JSON / CSV output mode**. Rejected — out-of-scope per roadmap.
  Machine-readable system info should consume mihi directly, not
  shoehorn into iam's display surface.

## References

- `docs/development/state.md` — current output sample.
- `docs/examples/sample-output.txt` — canonical sample.
- `docs/development/roadmap.md` — milestone arc; this ADR is M4.
- `CLAUDE.md` — load-bearing principles and hard constraints.
