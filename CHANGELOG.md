# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [1.0.0] — 2026-05-20

**Output-shape freeze. M6 closed.** iam reaches the v1.0 contract
in lockstep with mihi 1.0.0 shipping. Per CLAUDE.md *"v1.0, after
which the line order, label format, and exit codes are frozen"* —
ADR 0002's identity → runtime → hardware spine
(Distro / Host / Kernel / Uptime / CPU / GPU? / Memory) is the
written contract from this release onward. Any future change to
line order, label width, label spelling, `(unknown)` fallback, or
exit-code discipline is a real `Breaking` requiring a major-version
bump.

No source changes since v0.9.0 RC. The v0.9.0 cut deliberately
sat as the iam-side freeze candidate while we waited for mihi 1.0;
the v1.0.0 cut is the documented single-line `[deps.mihi]` repin
plus the milestone-closure docs/audit work.

### Changed
- `cyrius.cyml` — `[deps.mihi] tag` 0.7.0 → 1.0.0. mihi 1.0.0's
  bundle (`dist/mihi.cyr`) is module-content byte-identical to
  0.7.0 per mihi's own CHANGELOG; the only diff is the
  `# Version: 1.0.0` header stamp. iam's runtime output is
  byte-for-byte equal to v0.9.0 RC on archaemenid.
  `[deps.ai-hwaccel]` stays at 2.2.6 (mihi 1.0.0 pins the same
  transitive). `[package].cyrius` stays at 6.0.1 (matches mihi
  1.0.0's pin).
- `VERSION` — 0.9.0 → 1.0.0.

### Security
- `docs/audit/2026-05-20-v1.0.0-audit.md` — mandatory
  mihi-major-bump audit filed per the v0.9.0 audit's *Next audit
  trigger* clause. Verdict: pass. mihi 0.7.0 → 1.0.0 is a clean
  repin with frozen probe-API surface; F-001 stays closed, F-002
  carries as INFO (now formalized by mihi's own v1.0 contract
  freeze on cstring semantics). External CVE / 0-day research
  pass clean against the dep tree. Prior audit docs
  (v0.9.0 / M5.5 / M5) stay in the directory as historical record.

### Notes
- This is a **shape-and-contract freeze**, not a feature freeze.
  The roadmap's "Deferred" entries (network probes, runtime
  state — Battery / Swap / Disk) remain reopenable post-v1.0; any
  such addition that disturbs the v1.0 line spine becomes a real
  `Breaking` and requires a major-version bump from here.
- Dogfood (`~/.local/bin/iam` → `build/iam`, fired from `~/.zshrc`)
  auto-propagates the new build on the next interactive shell. No
  manual cutover needed.

## [0.9.0] — 2026-05-19

**M6 release candidate.** F-001 (TTY-escape sanitization at the
renderer boundary) lands as the v1.0 freeze prerequisite identified
across the M5 / M5.5 audits. v0.9.0 RC sits as the iam-side
release candidate while we wait for mihi 1.0 to ship; the v1.0 cut
will only need to repin mihi at that point.

### Security
- **F-001 mitigation** — `iam_copy_value` in `src/display.cyr`
  replaces any value-column byte `< 0x20` with `?` (0x3F).
  Neutralizes ANSI CSI escapes (lead byte 0x1B), TAB, line-injection
  attempts (LF / CR), and the rest of the C0 control range against
  hostile data arriving via mihi-returned strings (hostname, distro
  PRETTY_NAME, CPU model, kernel name+version, GPU name). Wired
  into `iam_render` (value path), `iam_render_buf` (valbuf path),
  and `iam_render_kernel` (name + version paths). Labels, padding,
  `IAM_UNKNOWN_TEXT`, and the trailing `\n` continue through plain
  `iam_copy` — they're iam-controlled. DEL (0x7F) and the C1
  control range (0x80–0x9F) are intentionally not sanitized to
  preserve legitimate UTF-8 byte sequences in distro PRETTY_NAMEs.
- `docs/audit/2026-05-19-v0.9.0-audit.md` — follow-up audit doc
  filed superseding M5.5 for the v0.8.0 reorder + v0.9.0 F-001
  scope. Verdict: pass. **F-001 closed**; F-002 carries as
  accepted-risk INFO. **No remaining v1.0 blockers on the iam
  side** — mihi 1.0 ship is the only external gate. M5 and M5.5
  audit docs stay in the directory as historical record per
  audit-trail convention.

### Added
- 15 new byte-exact assertions in `tests/iam.tcyr` under a new
  `test_group("display.cyr — F-001 TTY-escape sanitization")`.
  Cases: `iam_copy_value` boundary (0x1F → `?`, 0x20 unchanged,
  TAB → `?`); `iam_render` ESC-laden value; `iam_render`
  newline-injection attempt (proves the line-equals-one-fact
  contract holds); `iam_render_buf` ESC mid-buffer;
  `iam_render_kernel` ESC + TAB in name + version;
  `IAM_UNKNOWN_TEXT` fallback unchanged. Test inputs built via
  explicit `store8` (no reliance on Cyrius string-literal escape
  support beyond `\n`). Total assertion count 90 → 105.
- `iam_copy_value(out, cap, pos, src, len)` in `src/display.cyr`
  — the canonical value-column copy primitive. Same overflow
  contract as `iam_copy` (returns -1); single comparison per byte;
  cost is invisible at the bench scale.

### Changed
- `src/display.cyr` — three renderers swap value-side `iam_copy`
  calls to `iam_copy_value` (`iam_render` line 109; `iam_render_buf`
  line 137; `iam_render_kernel` lines 165 + 169). Plain `iam_copy`
  still backs labels, padding, the `(unknown)` fallback, and the
  trailing newline.
- `docs/benchmarks.md` — fourth row added to the trend table at
  v0.9.0 (M6 RC): **1510 µs median, 3 trials × N=500** on
  archaemenid. Sits inside the v0.5.0 noise floor (1490–1545 µs);
  sanitizer is invisible at this scale. Toolchain row bumped
  cyrius 6.0.0 → 6.0.1 (matches the active local pin and resolves
  the drift surfaced by the v0.7.0 doc-health scaffold). History
  entry added for the v0.9.0 re-measurement. ADR-reference comment
  updated 0001 §3 → 0001 §5 (carried over by 0002). "Three-point
  trend" header → "Four-point trend"; M3/M4 noise-floor commentary
  expanded to include v0.9.0 as the third in-band point.
- `VERSION` — 0.8.0 → 0.9.0.

### Notes
- The audit-trigger clause from the M5.5 audit fired correctly:
  the v0.8.0 reorder counted as "the first source change since
  v0.5.0," and the F-001 cut bundled the follow-up audit per the
  *Next audit trigger* schedule. The v0.9.0 audit is now the
  canonical iam-side artifact through to the v1.0 cut.
- No state.md / roadmap.md / doc-health.md content changes are
  flagged Breaking — the only Breaking change in the M6 path was
  the v0.8.0 reorder, which already shipped under the pre-v1.0
  grace. v1.0 itself locks the contract; from v1.0 onward, any
  output-shape change becomes a real `Breaking`.

## [0.8.0] — 2026-05-19

ADR 0002 accepted — output-shape reorder lands. Line order now
follows the **identity → runtime → hardware** spine
(**Distro / Host / Kernel / Uptime / CPU / GPU? / Memory**) instead
of the v0.3.0-era ordering that ADR 0001 codified. Optional `GPU:`
line now slots between CPU and Memory (keeping the hardware block
contiguous) instead of trailing at position 7. Done now while the
consumer count is zero — a v1.0+ change would have been a real
`Breaking` against a frozen contract.

### Breaking (pre-v1.0)
- **Output line order changed.** New canonical six-line shape:
  `Distro / Host / Kernel / Uptime / CPU / Memory`. New seven-line
  shape (when `mihi_gpu_count() > 0`): `Distro / Host / Kernel /
  Uptime / CPU / GPU / Memory`. Anyone parsing iam output by line
  position must update; label-based parsers (`grep ^Distro:` etc.)
  are unaffected. Per CLAUDE.md, output-shape changes remain
  `Breaking` until v1.0 freezes the contract.

### Added
- `docs/doc-health.md` — doc-currency ledger scaffolded following
  the cyrius / agnosticos / first-party convention. Scaled to
  iam's ~14-doc tree (vs cyrius's ~105). Bucket counts at the top,
  per-doc rows across six tiers (structural / architecture /
  development / ADRs / audits / guides+examples), refresh
  procedure, and the forward doc-policy commitment table. ADR 0002
  was the lone ❓ Open strategic question entry on the v0.7.0
  scaffold; closed in this cut by accepting the reorder.
- `CLAUDE.md` *Docs* index — pointer at the new `docs/doc-health.md`
  so the ledger is discoverable from the project entry doc.

### Changed
- `src/main.cyr` — `iam_render*` accumulation block reordered to
  the new sequence (`Distro / Host / Kernel / Uptime / CPU / GPU? /
  Memory`). Optional GPU line is now an inline conditional between
  CPU and Memory instead of a trailing tail-append, dropping the
  "always position 7" special case the M5 audit referenced. Probe
  calls at the top of `main()` did not move; only the order of
  `iam_render(&out + pos, …)` calls changed. ADR-reference comment
  updated 0001 → 0002.
- `tests/iam.tcyr` — 39 ADR-contract byte-exact assertions
  regenerated against the new order. Per-label padding tests
  (locked at ADR 0001 §1, which carries over) are unchanged.
  Seven-line test rebuilt from a fresh buffer rather than appending
  to the six-line buffer (GPU now mid-output, not at the tail).
  Total assertion count unchanged at 90.
- `docs/adr/0001-output-shape.md` — status `Accepted` →
  `Superseded by 0002 (2026-05-19)`. Header note marks §2-§3
  superseded and §1, §4, §5, §6 carry-over. Body left intact as
  historical record per ADR convention.
- `docs/adr/0002-output-shape-reorder.md` — status `Proposed` →
  `Accepted (2026-05-19)`. `Supersedes` field expanded to spell out
  §2-§3 only.
- `docs/adr/README.md` — index entry added for 0002; 0001 marked
  superseded.
- `docs/examples/sample-output.txt` — regenerated by running
  `./build/iam > docs/examples/sample-output.txt` on archaemenid
  against the new contract.
- `docs/development/state.md` — *Shape* section reworded for new
  order + ADR 0002 link; *Output* section sample regenerated + the
  layout-contract bullets updated to mention the
  identity → runtime → hardware spine and the new GPU mid-output
  position. Version section will refresh at the v0.8.0 cut.
- `VERSION` — 0.7.0 → 0.8.0.
- `.github/workflows/ci.yml` — Smoke run + DCE parity check
  expectations updated for the ADR 0002 line order. `want6` /
  `want7` strings rewritten (`Distro Host Kernel Uptime CPU Memory`
  for six-line; GPU slotted between CPU and Memory at position 6
  for seven-line). Required-label loops in both jobs now iterate
  in the new sequence. Comment block updated to reference ADR 0002
  + v0.8.0. Caught by the v0.8.0 push CI run.

### Notes
- M5 audit (`docs/audit/2026-05-19-audit.md`) and M5.5 audit
  (`docs/audit/2026-05-19-m5.5-audit.md`) source-review sections
  reference `main.cyr` line numbers that shifted slightly with the
  reorder. Findings (F-001, F-002) are unaffected — every probe is
  still bounded, every renderer return checked, exit-0 discipline
  intact. F-001 mitigation remains the lone v1.0 blocker; first
  source change since v0.5.0 will trigger the *Next audit trigger*
  clause regardless.

## [0.7.0] — 2026-05-19

M5.5 complete — security + code re-audit. Expanded audit gate
between v0.6.0 and the v0.9.0 RC: added a web-research pass for
0days / CVEs against the dep tree and a full source re-walk against
the M5 audit's findings checklist. v0.7.0 is the milestone-closure
cut; source `src/*.cyr` is unchanged from v0.6.0 (and from v0.5.0
before it). F-001 remains the lone v1.0-blocking item — its
mitigation will land separately ahead of the v0.9.0 RC.

### Added
- `docs/audit/2026-05-19-m5.5-audit.md` — refreshed audit
  superseding the M5 doc for the M5.5 scope. Verdict: **pass**, no
  new findings. Records the dep-tree CVE search (mihi 0.7.0,
  ai-hwaccel 2.2.6, cyrius 6.0.1, stdlib bundle — all first-party
  with no public CVE / advisory entries), notes the
  cyrius ≠ "Cyrus IMAP" / ai-hwaccel ≠ "NVIDIA Container Toolkit"
  name collisions so future auditors don't have to re-prove the
  disambiguation, and confirms zero source drift since the v0.5.0
  audit baseline via `git diff a57c17b..HEAD -- src/`. M5 doc stays
  in the directory as historical record per audit-trail convention.
  The `-m5.5-` infix on the filename disambiguates the same-day
  collision with the M5 audit.

### Changed
- `docs/development/roadmap.md` — M5.5 marked fully shipped (all
  three deliverables ✅). M6 (v0.9.0 RC + v1.0) remains gated on
  F-001 mitigation and mihi 1.0.
- `docs/development/state.md` — version bumped 0.6.0 → 0.7.0;
  *Audit* section updated to point at the new M5.5 doc; *Next*
  section trimmed to F-001 mitigation + M6 (M5.5 closed).
- `VERSION` — 0.6.0 → 0.7.0.

## [0.6.0] — 2026-05-19

M5 complete — harden + dogfood. All four M5 deliverables landed
against the v0.5.0 codebase; v0.6.0 is the milestone-closure cut.
Code unchanged from v0.5.0; everything below is docs, audit, and
process artifacts.

### Added
- `docs/benchmarks.md` — invocation-time benchmark methodology and
  three-point trend (v0.3.0 → M3@9df0859 → v0.5.0). M5 < 10 ms
  cold-start gate verified: ~1.5 ms on archaemenid, 6.5× headroom.
  Trend identifies the M3 GPU probe as the dominant cost and
  records that the M4 single-flush refactor was nominal at this
  scale.
- `docs/audit/2026-05-19-audit.md` — P(-1) security audit pass
  against v0.5.0. Verdict: pass with one open finding (F-001:
  TTY-escape sanitization on mihi-returned strings, LOW — gates
  M6 freeze) and one INFO note (F-002: `strlen` cstring
  invariant cross-referenced to mihi audit).
- `docs/adr/0002-output-shape-reorder.md` — **Proposed** ADR for
  a fastfetch-similar top-down line order (Distro → Host →
  Kernel → Uptime → CPU → GPU? → Memory). Supersedes ADR 0001 §2
  and §3 if accepted; ADR 0001 §1, §4, §5, §6 carry over
  unchanged. Decision deferred to a later cut; v0.6.0 ships with
  ADR 0001's order intact.
- Dogfood wiring: `~/.local/bin/iam` → `build/iam` symlink,
  `~/.zshrc` calls `iam` on every interactive shell. Ran clean
  from v0.5.0 cut through v0.6.0 cut across the maintainer's
  full set of terminal sessions — no regressions, no shell
  startup degradation worth complaining about (~9 ms total
  startup with starship + iam vs ~7 ms without iam).

### Changed
- `docs/development/roadmap.md` — M5 marked fully shipped (all
  four deliverables ✅). Telescoped the release plan: M5
  acceptance now cuts as v0.6.0 (was v0.9.0). Added **M5.5**
  milestone (Security + code re-audit, v0.7.0) between M5 and
  M6 — expanded audit with web research against the dep tree
  for 0days / CVEs. M6 now cuts v0.9.0 RC before v1.0.
- `docs/development/roadmap.md` — "Out of scope (for v1.0)"
  section restructured. Split into "Not iam's job (use the
  right tool)" and "Deferred (may reopen later)" — the old
  flat "never / forever" framing conflated identity constraints
  with sequencing decisions. Each entry now names the right
  alternative tool (or the gating reason for the deferral).
- `docs/development/roadmap.md` — ASCII-logos entry expanded to
  hand the reader three concrete escape valves (BannerManor,
  neofetch / fastfetch, GPL-3.0 fork) instead of just one.
- `docs/development/roadmap.md` — Color / theming entries split
  honestly: iam *being* a theming engine is identity-rejected,
  iam *consuming* the user's shell theme is a real post-v1.0
  conversation.
- `docs/development/state.md` — version bumped 0.5.0 → 0.6.0;
  added *Benchmarks* and *Audit* sections; *Next* lists M5.5,
  F-001 mitigation, and M6 in sequence.
- `VERSION` — 0.5.0 → 0.6.0.
- `cyrius.cyml` — toolchain pin bumped 6.0.0 → 6.0.1 to track
  the active local cycc; build + 90-assertion test suite
  identical under the new toolchain (no observable behavior
  change, drift warning resolved).

## [0.5.0] — 2026-05-19

M4 complete — output-shape ADR landed and locked into executable
form via byte-exact tests. The bytes iam emits are now a written
contract codified across three surfaces: the ADR doc, the canonical
sample, and the test suite. M4's three deliverables shipped in one
cut.

### Added
- `docs/adr/0001-output-shape.md` — accepted ADR locking line order,
  8-byte label column, `(unknown)` fallback policy, the
  single-optional-GPU-line rule, no-color / pipe-equivalence
  guarantee, exit-0 contract. Includes the alternatives-considered
  trail (tab separators, variable label width, color-on-TTY,
  per-GPU lines, JSON output).
- `docs/examples/sample-output.txt` — canonical sample output
  captured on archaemenid. Referenced from the ADR.
- ADR index updated in `docs/adr/README.md`.
- 39 byte-exact assertions in `tests/iam.tcyr` covering every ADR
  §1–§4 case: per-label padding for all seven labels (CPU/Memory/
  Kernel/Host/Distro/Uptime/GPU), `(unknown)` fallback paths
  through every renderer variant, full six-line and seven-line
  assembled output shapes, full all-probes-failed degraded shape.
  Test count: 51 → 90.

### Changed
- **Renamed renderer family**: `iam_emit` / `iam_emit_buf` /
  `iam_emit_kernel` → `iam_render` / `iam_render_buf` /
  `iam_render_kernel`. Renderers now write into a caller-supplied
  byte buffer and return bytes-written (or `-1` on overflow)
  instead of issuing per-line `write(2)` syscalls. This makes the
  ADR contract observable to tests.
- **Single-flush driver**: `src/main.cyr` now accumulates the full
  seven-line output into a 4 KiB stack buffer and emits it with
  one `print(buf, len)` call. Side benefit: fewer syscalls per
  invocation (one vs ~14 previously) — incidental win against the
  M5 < 10 ms cold-start target.
- `iam_render*` helpers consolidate byte-append + bounds-check via
  two new internal primitives (`iam_put`, `iam_copy`), keeping each
  renderer body ~15 lines of comprehensible logic.

### Compatibility
- The renderer rename is iam-internal — no external consumers
  (iam is a binary, not a library). The output bytes are
  unchanged; `./build/iam` on the same host emits a byte-identical
  six-line or seven-line report at v0.4.0 vs v0.5.0.

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
