# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.3.0] — 2026-05-19

M1+M2 in one bite: mihi was already at 0.7.0 (past M3's gate), so
this release ships six display lines instead of staging M1 (three
lines) and M2 (three more) across two cuts.

### Added
- `[deps.mihi]` pinned to 0.7.0 — every displayed line routes through
  a `mihi_*` probe (CLAUDE.md "probe via mihi, never inline" rule).
- `[deps.ai-hwaccel]` pinned to 2.2.6 — pulled in transitively via
  `mihi/gpu.cyr` references inside `dist/mihi.cyr`. DCE drops the
  unused GPU surface from the linked binary; the parser still
  resolves the symbols at build time.
- `src/display.cyr` — line emitter (`iam_emit` / `iam_emit_buf` /
  `iam_emit_kernel`), byte-size formatter (`iam_format_bytes`,
  binary units, floor-rounded), shared `iam_uint_into` helper.
- `src/uptime.cyr` — seconds → "1d 2h 3m" formatter with zero-field
  elision and a `<1m` floor for sub-minute / fresh-boot displays.
- Six-line output: CPU model · Memory total · Kernel name+version ·
  Host (hostname) · Distro (PRETTY_NAME, ID fallback) · Uptime.
- Unknown-probe fallback: null/negative mihi returns render as
  `(unknown)` in the value column (never an empty line, never a
  stderr error — iam is a presentation surface).
- Tests covering formatter logic — 51 assertions across
  `iam_uint_into`, `iam_format_bytes`, `iam_format_uptime`,
  including boundary, floor, cap-too-small, and preview-shape cases.

### Changed
- `src/main.cyr` — replaced scaffold "iam v0.1.0 — scaffold" stub
  with the six-probe driver. Includes `src/display.cyr` +
  `src/uptime.cyr`; mihi/ai-hwaccel bundles auto-include via
  `cyrius.cyml [deps.*]`.
- `[deps].stdlib` widened to the union of mihi's and ai-hwaccel's
  bundle needs (adds `slice`, `agnosys`, `fs`, `tagged`, `process`,
  `fnptr`, `thread`, `freelist`, `hashmap`, `ct`, `json` on top of
  the scaffold's set). DCE keeps the binary lean.

### Breaking (pre-v1.0)
- Output shape changed from one scaffold line to six labeled lines.
  Per CLAUDE.md: output-shape changes remain `Breaking` until v1.0
  freezes the contract at the M4 ADR.

## [0.1.0]

### Added
- Initial project scaffold
