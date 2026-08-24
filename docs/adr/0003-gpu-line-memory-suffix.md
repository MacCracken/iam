# 0003 — GPU line carries on-device memory

**Status**: Accepted (2026-08-01)
**Date**: 2026-08-01
**Extends**: [0002](0002-output-shape-reorder.md) §3 (optional-line policy). Line order, label column, `(unknown)` fallback, no-color, and exit-0 all carry over untouched — 0002 remains the shape contract; this ADR only decides what goes in the GPU line's value column.

## Context

`mihi` has exposed three GPU probes since iam's M3 cut:
`mihi_gpu_count()`, `mihi_gpu_name(idx)`, and
`mihi_gpu_memory_bytes(idx)`. iam has consumed the first two since
v0.4.0 and ignored the third. The GPU line has therefore always
read:

```
GPU:    AMD Radeon (PCI 0x1002:0x1638)
```

— an identity with no capacity. Meanwhile the `Memory:` line right
below it answers exactly that question for system RAM. A reader
scanning the hardware block gets "which GPU" and "how much RAM" but
not "how much VRAM", which is the one figure that decides whether a
model fits on the box. On the maintainer's archaemenid that figure
is 3 GiB, and iam was the only line of the hardware block declining
to state its own.

Two things made this the moment to close the gap rather than a
neofetch-style embellishment we'd have rejected under the
feature-creep gate:

1. **AGNOS now reports it.** Kernel syscall **#89 `gpu_caps`** grew
   an opt-in third size tier (`len >= 96`) carrying GPU identity:
   `vendor_id`, `device_id`, **`vram_mb`**, and a 16-byte ASIC tag.
   AGNOS has known its accelerator's identity since probe and never
   offered it to ring 3. The kernel work is done; mihi bridges
   #89 into `mihi_gpu_*`; iam is the last layer, and if iam prints
   only the name then `vram_mb` — the one genuinely new number in
   the new ABI tier — dies one call short of a human.
2. **The two sources agree.** Linux sysfs
   (`/sys/class/drm/card1/device/mem_info_vram_total`) reads
   3221225472 on archaemenid; the #89 `vram_mb` field reads 3072 on
   the same silicon. Same 3 GiB, so the line reads identically on
   Linux and on AGNOS. The suffix is not an AGNOS-only affordance
   bolted onto a Linux tool.

The countervailing pressure is CLAUDE.md's load-bearing *keep it
whoami-simple* rule and the post-v1.0 freeze. Neither forbids this:
the freeze enumerates **line order, label width, label spelling,
`(unknown)` fallback, and exit-code discipline**, and this ADR moves
none of them. `state.md`'s post-v1.0 stewardship clause names the
exact path — *"an addition that fits cleanly alongside the spine
needs ADR justification and re-audit but can ship in a `Minor`
cut."* This is that addition, and this is that ADR.

## Decision

When an accelerator is detected **and** its on-device memory is
known, the GPU line's value column is the device name followed by a
space and the size in square brackets:

```
GPU:    AMD Radeon (PCI 0x1002:0x1638) [3 GiB]
```

Scope — what is in:

1. **First device only.** Unchanged from ADR 0002 §3. iam emits one
   `GPU:` line carrying `mihi_gpu_name(0)` and
   `mihi_gpu_memory_bytes(0)`. The six-or-seven-line count guarantee
   is preserved exactly.
2. **Brackets, not parens.** mihi device names already carry a
   parenthesised PCI id, so a parenthesised suffix would read as a
   second, nested id: `AMD Radeon (PCI 0x1002:0x1638) (3 GiB)`.
   Brackets keep the two facts visually distinct at a glance.
3. **Same units as `Memory:`.** The suffix routes through the
   existing `iam_format_bytes` — binary units, integer floor, no
   fractional part. 3 GiB of VRAM and 59 GiB of RAM are formatted by
   one function, so the hardware block is internally consistent.
4. **Silent degrade, no placeholder.** `mihi_gpu_memory_bytes`
   returns `0 - 1` for an out-of-range index and `0` where the
   backend carries no size figure. Both fall below the driver's
   `> 0` gate, and the line renders as the bare device name —
   **byte-for-byte what iam emitted before this ADR**. The suffix is
   itself optional; an optional element does not earn an
   `(unknown)`. §4 of ADR 0001 governs *required* value columns, and
   the device name remains the required one.
5. **Name still load-bearing.** `mihi_gpu_name(0) == 0` renders
   `GPU:    (unknown)` regardless of the size. A capacity with no
   identity is not a fact worth a line.

Scope — what is out:

- **No new line.** The size rides in the existing GPU line's value
  column. A `VRAM:` line would grow the label set and break the
  six-or-seven-line count guarantee.
- **No multi-GPU expansion.** See *Alternatives considered*.
- **No vendor / device id / ASIC tag in the output.** #89 also
  carries `vendor_id`, `device_id`, and a `gfx90c/DCN2.1`-style ASIC
  tag. Those are mihi's to fold into the device *name* if it wants
  them; iam prints whatever string mihi hands it and does not
  compose identity itself.

Implementation: a fourth renderer, `iam_render_gpu` in
`src/display.cyr`, alongside `iam_render` / `iam_render_buf` /
`iam_render_kernel`. It takes the size as a caller-formatted
`buf + len` — the same convention `iam_render_buf` uses for
`Uptime:` and `Memory:` — so the renderers keep laying out bytes and
the driver keeps owning formatting. Device-name bytes route through
`iam_copy_value`, so the F-001 sanitizer covers the composed line
exactly as it covered the bare one.

## Consequences

- **Positive** — the hardware block finally answers the capacity
  question for every line in it. CPU says what, GPU says what and
  how much, Memory says how much.

- **Positive** — the new AGNOS `gpu_caps` size tier reaches a human.
  `vram_mb` was added to syscall #89 specifically so ring 3 could
  see it; without this ADR the field is exposed and unread.

- **Positive** — the degrade path is a strict superset of the old
  behavior. Every host where the size is unknown emits the exact
  bytes it emitted at v1.1.5, which bounds the blast radius of the
  change to hosts that gained information.

- **Negative** — the GPU value column is now composite. It is the
  second line to compose two facts (`Kernel:` joins name and version
  and has since v0.3.0), but it is the first whose second fact is
  *conditional*, so `tests/iam.tcyr` now has to lock two assembled
  seven-line shapes instead of one.

- **Negative** — a consumer that string-matched the full GPU value
  against a device name will see a trailing ` [3 GiB]` it did not
  before. `state.md` records zero consumers, and label-based
  matching (`grep ^GPU:`) plus line-count parsing are both
  unaffected — but this is a real, if small, value-column change to
  a post-v1.0 tool and is recorded here as such rather than waved
  through.

- **Neutral** — the line grows by at most 11 bytes (`" [1023 PiB]"`).
  `IAM_OUT_CAP` is 4096 against a worst case near 500; no cap change
  needed.

- **Neutral** — triggers the `state.md` re-audit clause (any
  `src/*.cyr` change beyond comments). Audit filed alongside this
  ADR.

## Alternatives considered

- **Print nothing; ship the row as-is.** Rejected. The GPU row
  already existed and already renders on Linux — the honest reading
  of "surface the accelerator" is that the row appearing on AGNOS is
  mihi's win, not iam's, and iam's own contribution to the chain is
  precisely the number it was throwing away. Doing nothing would
  have left `vram_mb` exposed by the kernel, bridged by mihi, and
  dropped on the floor by the presentation layer.

- **A separate `VRAM:` line.** Rejected. Grows the label set (which
  the freeze covers), breaks the six-or-seven-line count guarantee
  that ADR 0002 §3 explicitly preserved, and splits one device's
  facts across two lines against the one-fact-per-line reading of
  "what accelerator is in this box".

- **One line per accelerator on multi-GPU hosts.** Rejected — this
  is the option the freeze actually forbids. ADR 0002 §3 guarantees
  output is six or seven lines; N GPU lines makes it six-plus-N and
  breaks every line-count parser. ADR 0002 and the M3 roadmap entry
  both recorded first-device-only as a deliberate whoami-simple
  choice, and nothing in the AGNOS work changes that reasoning.
  Consumers needing per-device detail call mihi directly — which is
  exactly what iam's "thin presentation layer" framing means.

- **A count suffix on the single line** (`… [3 GiB] (+1 more)`).
  Rejected. It preserves the line count but spends the value column
  on bookkeeping about facts iam has decided not to print — the
  worst of both options, and it reads like an apology.

- **Parenthesised suffix** (`AMD Radeon (PCI 0x1002:0x1638) (3 GiB)`).
  Rejected on the concrete archaemenid string: mihi's device name
  already ends in a parenthesised id, so a second parenthesised
  group reads as nested or duplicated. Brackets are unambiguous
  against every device name mihi currently produces.

- **Render `(unknown)` in the suffix when the size is missing**
  (`AMD Radeon [(unknown)]`). Rejected. `(unknown)` exists so a
  *required* line never vanishes and never blanks — the suffix is
  optional, so silence is the correct absence signal, and it keeps
  the no-size output byte-identical to v1.1.5.

- **Decimal precision** (`3.0 GiB`). Rejected — `iam_format_bytes`'s
  integer-floor contract is settled (ADR 0001, `display.cyr`
  docstring: *"the user wanted 'roughly how much RAM', not a
  measurement"*). VRAM is not a special case.

## References

- [ADR 0001](0001-output-shape.md) §4 — `(unknown)` fallback policy;
  scoped to required value columns.
- [ADR 0002](0002-output-shape-reorder.md) §3 — optional-line
  policy, first-device rule, six-or-seven-line count guarantee. This
  ADR extends §3's value column and changes none of its rules.
- `src/display.cyr` — `iam_render_gpu`.
- `src/main.cyr` — probe + format + gate.
- `docs/development/state.md` — post-v1.0 stewardship clause that
  authorizes an additive `Minor` cut with ADR justification.
- AGNOS kernel syscall **#89 `gpu_caps`**, `len >= 96` tier —
  `vendor_id` / `device_id` / `vram_mb` / ASIC tag. The upstream
  source of the number this ADR prints.
