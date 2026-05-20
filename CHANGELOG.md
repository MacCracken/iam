# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [0.5.0] — 2026-05-19

M4 — output-shape ADR. The bytes iam emits are now a written
contract, not just folklore. No source code changes; this release
codifies decisions already shipped in v0.3.0 and v0.4.0 so they
survive a v1.0 freeze.

### Added
- `docs/adr/0001-output-shape.md` — accepted ADR locking line order,
  8-byte label column, `(unknown)` fallback policy, the
  single-optional-GPU-line rule, no-color / pipe-equivalence
  guarantee, exit-0 contract. Includes the alternatives-considered
  trail (tab separators, variable label width, color-on-TTY,
  per-GPU lines, JSON output).
- `docs/examples/sample-output.txt` — canonical sample output
  captured on archaemenid. Referenced from the ADR; the seed for
  the byte-exact mihi-mock tests that follow in a later bite.
- ADR index updated in `docs/adr/README.md`.

### Deferred
- Byte-for-byte tests against a fixed mihi-mock input — the third
  M4 deliverable per the roadmap. Requires either routing
  `iam_emit` writes through a buffer or building a mock-mihi shim;
  taken as a separate bite to keep this release pure-documentation.

## [0.4.0] — 2026-05-19

M3 — GPU line. Single trailing `GPU:` line driven by `mihi_gpu_*`,
suppressed entirely when `mihi_gpu_count()` is 0. Multi-GPU systems
show the first device only; consumers who need per-device detail
call mihi directly (whoami-simple rule: iam picks one line per fact).

### Added
- `GPU:` line in the documented output, emitted after `Uptime:` when
  `mihi_gpu_count() > 0`. Pulls the device name from `mihi_gpu_name(0)`
  (ai-hwaccel's no-exec accelerator registry, masked of all subprocess-
  spawning backends — pure sysfs reads).
- CI smoke gate widened to accept 6 or 7 lines; the required label
  set (CPU/Memory/Kernel/Host/Distro/Uptime) is unchanged.
- DCE parity check now cross-checks GPU-line presence between
  non-DCE and DCE'd builds — a DCE that dropped a live
  `mihi_gpu_*` call path would diverge here.

### Notes
- CI hosted runners (ubuntu-latest) have no accelerator, so the GPU
  line is exercised only on self-hosted / dev-machine runs. The DCE
  parity check still catches drops because it compares the two
  builds against each other, not against an expected fixed count.

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
