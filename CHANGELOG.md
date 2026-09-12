# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

## [1.1.8] - 2026-09-11

### Changed

- **Toolchain `6.5.35` → `6.6.2`.** Migrated to the `Result` value form:
  1 first-party file(s) changed. Every surface re-verified — build, tests, and any
  bench/fuzz/distlib target the repo ships.


## [1.1.7] — 2026-08-23

**P(-1) hardening sweep. Five findings, one of them a live contract violation.**
Full write-up in [`docs/audit/2026-08-23-v1.1.7-audit.md`](docs/audit/2026-08-23-v1.1.7-audit.md).

The one that matters: **iam could silently print five lines instead of six.** ADR 0002
guarantees the six required labels always appear and CI gates on it, but nothing bounded a
probe value against the 4 KiB buffer it had to fit into. On overflow `iam_render` returned
its error sentinel, the driver's `if (n > 0) { pos = pos + n; }` left the cursor where it
was, and the whole `CPU:` line vanished — no error, no stderr, just a missing fact.
Reproduced with a 5000-byte CPU model, fixed, and locked with a test that fails against the
v1.1.6 source.

Runtime output on archaemenid is **byte-identical to v1.1.6**. Every fix here changes
behaviour only for input that was already pathological.

### Added

- **`IAM_VALUE_MAX` (256 bytes)** — a hard cap on any single value column, enforced inside
  `iam_copy_value`, the one choke point all four renderers route their value through. An
  over-long value now truncates and **the line still renders**. This is what CLAUDE.md's
  "even mihi output gets bounds-checked at display time" was asking for and iam was not
  doing: the driver hands mihi an 8 KiB scratch for the CPU model alone, against a 4 KiB
  buffer for all seven lines. Worst-case seven-line output with every value clamped is
  1673 bytes.
- **UTF-8-safe truncation** — the clamp backs off while the byte at the cut is a
  continuation byte (0x80–0xBF), so a truncated value ends on a character boundary instead
  of emitting half a sequence. (`load8` was verified unsigned before relying on that test.)
- **`iam_flush`** — a short-write- and `-EINTR`-safe stdout flush, replacing the unchecked
  single `print()`.
- **`IAM_EINTR`**, defined locally rather than taken from the stdlib — see *Fixed*.
- **19 assertions** in `tests/iam.tcyr` (122 → **141**): the six-line contract under a
  pathological value, the bound holding across all four renderers, a value exactly at the
  cap, UTF-8 boundary back-off, DEL sanitization, UTF-8 bytes surviving sanitization, and
  the copy-primitive bounds.

### Fixed

- **F-003 (HIGH) — an over-long probe value dropped a required line.** Detail above. The
  reason four previous audits missed it is worth recording, and it is not that the threat
  was unimagined. `SECURITY.md` has named it since the first audit: *"if mihi returns
  adversarially-**large** or malformed buffers […] iam's display formatter must still bound
  writes."* Writes **were** bounded the whole time — `iam_copy`'s capacity check never let a
  byte past the end, and no overflow ever occurred. What nothing verified was what iam
  *prints* once the bound is hit. Every audit confirmed iam does not overflow; none asked
  what it emits when it declines to. F-002's standing framing ("iam trusts mihi's NUL
  invariant") compounded it by anchoring each review on *malformed* input, when F-003 needs
  nothing malformed — a correctly NUL-terminated 5000-byte string is enough.
  Reachability is a crafted `/etc/os-release` `PRETTY_NAME`, a doctored `/proc` mount, or
  root — real configurations, and the failure is silent. Growing `IAM_OUT_CAP` was
  considered and rejected: it does not fix the class, and a 16 KiB stack buffer would blow
  the ~12 KiB agnos user-stack budget v1.1.5 was cut to respect.
- **F-004 (MEDIUM) — the report was flushed with a single unchecked `write(2)`.**
  `print()` discards `file_write`'s return, which discards `sys_write`'s, which is a bare
  `syscall(SYS_WRITE, ...)`. A short write silently truncated the report; an `-EINTR` lost
  it. Exact symmetric counterpart of the short-read defect mihi's 1.2.3 audit filed as A-1,
  on the login path where a half-printed MOTD is visible. `iam_flush` now loops until the
  kernel has taken every byte, retries `-EINTR`, and stops on any other negative return
  (EPIPE, EIO — nowhere useful to report it, and ADR 0001 §6 keeps exit at 0).
  **Verified not to hang**, which is the risk a retry loop introduces: early-closed pipe
  (`iam | head -c 10`) and fully closed stdout both exit 0 immediately.
- **`EINTR` is not portable to the agnos target.** The flush first used the stdlib
  constant, which compiles on host and `--aarch64` and **fails `--agnos`** — `EINTR` is
  declared in the Linux and macOS syscall modules but not in `syscalls_x86_64_agnos.cyr`.
  iam is an agnos build target, so it now defines `IAM_EINTR` locally. Caught by the
  cross-target build; a host-only gate would have shipped it.
- **F-005 (LOW) — the value sanitizer missed DEL (0x7F).** By ECMA-48 the C0 set is
  0x00–0x1F **plus** DEL, so DEL passed a filter whose stated intent is "no control bytes in
  the value column". Now `b < 32 || b == 127`. **C1 (0x80–0x9F) stays deliberately
  unfiltered** — general terminal-sanitization guidance says to strip it, and that guidance
  is wrong here: on a UTF-8 terminal those bytes are only ever multibyte continuation bytes,
  so filtering them would corrupt legitimate non-ASCII values while protecting against
  nothing. Recorded so a future audit does not "fix" it.
- **F-006 (LOW) — `iam_append` was dead code**, defined and never called, in a 150-line file
  whose stated value is being small and auditable. Its slot is now `iam_flush`.
- **F-007 (LOW) — the copy primitives accepted a negative length and silently rewound the
  write cursor.** `if (pos + len > cap)` passes for `len = -1`; the copy loop does not run;
  the function returns `pos + len`, moving the cursor *backwards* so the next write clobbers
  a byte of the previous line. Bound is now the non-wrapping `len > cap - pos` plus an
  explicit `len < 0` refusal. The first version of this regression test **passed against the
  unfixed source** — at `pos == 0` the buggy `0 + (-1)` is `-1`, the right answer by
  coincidence. With the cursor advanced, v1.1.6 returns **9**. The test uses a non-zero
  `pos` for exactly that reason.

### Changed

- **Formatting normalized** in `src/main.cyr` and `tests/iam.tcyr` — continuation lines used
  aligned-to-open-paren indentation where canonical style is 2 spaces per open paren, a
  divergence 6.5.35's formatter flags and 6.2.37's accepted. Whitespace-only: `diff -w` is
  empty for both, `cyrius fmt` is idempotent on the result, and 141/0 passes before and
  after. Deferred twice in the 1.1.6 cycle on readability grounds; mihi's own 1.2.2
  normalization settles the precedent and this closes the last cleanliness gap.

### Verified

- `cyrius build` **OK** on x86_64, `--agnos`, and `--aarch64`; `cyrius lint` **0 warnings**;
  `cyrius fmt --check` **clean** across all `src/` and `tests/`; `cyrius test` **141/0**.
- **Every finding's regression test was confirmed red against the v1.1.6 source** before
  being accepted as green — built from `git archive HEAD` into a scratch tree. A test that
  would also pass when the code is broken is worthless.
- Runtime byte-identical to v1.1.6 on archaemenid; stderr 0 bytes; TTY, pipe, and file all
  produce the same 194 bytes.
- **No runtime regression** — interleaved A/B, compiler held constant, three N=500 trials
  each: **1580 µs** median at v1.1.6 vs **1573 µs** at v1.1.7. The added per-byte comparison
  and clamp check are invisible at this scale; < 10 ms gate holds with ~6.4x headroom.

### Notes

- **`Minor`, not `Breaking`.** Line order, label set, label width, the `(unknown)` policy,
  and exit-0 are untouched. F-003 does not change the contract — it makes iam start
  honouring a guarantee ADR 0002 already made and the code could violate.
- **F-002 (`strlen` trust-dependence) still carries as INFO.** F-003's clamp bounds how much
  of an over-long value is *copied*; it does not bound the `strlen` scan, which still runs
  to the first NUL. Closing F-002 needs an `n`-limited probe variant from mihi — a mihi API
  question, not an iam one.

## [1.1.6] — 2026-08-23

**Two cuts in one: the GPU line learns how much memory the accelerator has, and the
whole dependency floor moves up.** The output change is the headline; the refresh under
it is what makes the headline reachable on more hosts, because mihi 1.2.4 is the release
that gives an aarch64 box a real `CPU:` value instead of `(unknown)`.

**The GPU line reports how much memory the accelerator has.**
iam has consumed `mihi_gpu_count()` and `mihi_gpu_name()` since v0.4.0 and ignored
`mihi_gpu_memory_bytes()` the whole time — the hardware block said which GPU but not how
much VRAM, the one figure `Memory:` answers for system RAM one line below. Closing that
gap is also what carries the AGNOS kernel's new `gpu_caps` **`vram_mb`** field
(syscall **#89**, the opt-in `len >= 96` identity tier) all the way to a human: the kernel
exposes it, mihi bridges it, and iam was the layer dropping it on the floor.

### Added

- **`iam_render_gpu` renderer** (`src/display.cyr`) — fourth renderer alongside
  `iam_render` / `iam_render_buf` / `iam_render_kernel`. Lays out
  `GPU:    <name> [<size>]`, taking the size as a caller-formatted `buf + len` (the same
  convention `iam_render_buf` uses for `Uptime:` and `Memory:`, so the renderers keep
  laying out bytes and the driver keeps owning formatting). Device-name bytes route
  through `iam_copy_value`, so the F-001 sanitizer covers the composed line exactly as it
  covered the bare one.
- **[ADR 0003](docs/adr/0003-gpu-line-memory-suffix.md)** — the GPU value column's
  contract: first device only, brackets not parens, `iam_format_bytes` units, silent
  degrade. Extends ADR 0002 §3's value column; ADR 0002's line order, label set, and
  six-or-seven-line count guarantee are untouched.
- **17 assertions** in `tests/iam.tcyr` (105 → **122**): the `iam_render_gpu` group
  (suffix shape, the real archaemenid parenthesised-PCI-id name, both degrade paths,
  `name = 0` with and without a size, overflow, ESC sanitization on the composed line)
  plus a second assembled seven-line case locking the size-unavailable shape.

### Changed

- **`GPU:` value column gains a bracketed size suffix when the probe knows the size**
  (`src/main.cyr`). On archaemenid the line moves from
  `GPU:    AMD Radeon (PCI 0x1002:0x1638)` to
  `GPU:    AMD Radeon (PCI 0x1002:0x1638) [3 GiB]`. Brackets rather than parens because
  mihi device names already end in a parenthesised PCI id. Formatting routes through the
  existing `iam_format_bytes`, so VRAM and RAM use identical binary-floor units.
- **Silent degrade preserves the old bytes exactly.** `mihi_gpu_memory_bytes` returns
  `0 - 1` for an out-of-range index and `0` where the backend has no size figure; both
  fall below the driver's `> 0` gate and the line renders as the bare device name —
  byte-for-byte what v1.1.5 emitted. No `(unknown)` placeholder: `(unknown)` exists so a
  *required* value column never blanks, and this suffix is optional. `mihi_gpu_name(0) == 0`
  still renders `GPU:    (unknown)` regardless of the size.
- **Toolchain pin `6.2.37` → `6.5.35`** (`cyrius.cyml [package].cyrius`) — closes the
  wrapper/manifest drift the installed toolchain had been reporting (`manifest-pin:
  6.2.37 (drift — wrapper is 6.5.35)`). CI reads the pin out of the manifest, so no
  workflow YAML changes (CLAUDE.md: never hardcode toolchain versions in CI).
- **`[deps.mihi]` `1.2.1` → `1.2.4`.** Probe API unchanged — every one of the eleven
  `mihi_*` symbols iam calls has the same signature and the same error sentinels, and
  the rendered output is byte-identical on archaemenid. What the span buys iam is
  upstream hardening it consumes for free: **1.2.3**'s A-1 fix (the `/proc` + `/sys`
  read path could return a *plausible wrong value* on a short read or `-EINTR` rather
  than an error — measured, not theoretical: a single `read` on `/proc/cpuinfo` returns
  3288 of the 8192 bytes iam asks for) and **1.2.4**'s D-1 fix, a device-tree
  `CPU:` source for aarch64 Linux where the probe previously returned nothing.
- **`[deps.ai-hwaccel]` `2.2.6` → `2.3.18`**, matching mihi 1.2.4's own transitive pin.
  Twelve upstream releases including three symbol de-collisions (`ERR_*` → `HWA_ERR_*`,
  `registry_new` → `hw_registry_new`, `BACKEND_COUNT` → `AIHW_BACKEND_COUNT`). None
  reach iam — it references no ai-hwaccel symbol directly; the bundle is present only so
  mihi's `gpu.cyr` parses.
- **`sakshi` added to `[deps] stdlib`.** ai-hwaccel 2.3.x routes its detect-path
  diagnostics through it, and the bundle is one concatenation, so the parser needs the
  module in scope even though iam logs nothing. This is not a guess: mihi 1.2.2 ships a
  `dist/mihi.deps` sidecar declaring the required fold, and iam's list is now that
  sidecar exactly — 21 modules, no drift.
- **`lib/` re-vendored at the 6.5.35 snapshot** (`cyrius lib sync --full`) — 108 `.cyr`
  files, clearing the `./lib/ shadows version-pinned .../6.5.35/lib — 10 bundled lib(s)
  differ` warning that the pin bump surfaced (`ganita`, `niyama`, `sigil`, `sandhi`,
  `yukti`, `patra`, `vani`, `mabda`, `sankoch`, `yantra` were all still at their
  6.2.22-era versions). New to the tree with this snapshot: the `lib/unicode/`
  sub-package (7 files) plus `async_macos`, `async_win`, `thread_macos`. iam links none
  of them; DCE drops them from the binary.

### Removed

- **Ten orphaned vendored modules pruned from `lib/`** — files absent from the 6.5.35
  snapshot *and* undeclared in `[deps] stdlib`, so nothing could include them:
  `agnosys-core.cyr` (left behind when the `[deps.agnosys]` git dep was dropped at
  cyrius 6.2.37), `base64` / `bigint` / `csv` / `cyml` / `json` / `toml` / `u128`
  (folded into the `bayan` distribution in the 6.2.x reorg) and `linalg` / `matrix`
  (folded into `ganita`). Verified unreferenced by `src/`, `tests/`, `cyrius.cyml`, and
  both dep bundles before removal; build, lint, tests, and runtime output are unchanged
  without them. Same prune mihi did at its 1.2.2 cut — `lib/` now matches the pinned
  snapshot exactly, plus the two dep bundles. `cyrius.lock` stays at **110** hash
  entries — a coincidence worth spelling out, since the count alone would suggest
  nothing moved: ten orphans left and ten new snapshot files arrived (`async_macos`,
  `async_win`, `thread_macos`, and the seven `lib/unicode/` files).

### Fixed

- **CI installed the toolchain by hand instead of using the installer, and that is why
  the `lib/` drift above went unnoticed for three cuts.** `.github/workflows/ci.yml` did
  `curl` the release tarball, `tar xzf`, then `cp` `bin/` and `lib/` into
  `$HOME/.cyrius/`. That populates `$HOME/.cyrius/{bin,lib}` but creates **no
  `$HOME/.cyrius/versions/<v>/` snapshot** — and the versioned snapshot is what `cyrius
  lib sync` reads from and what the compiler diffs `./lib/` against to emit
  `./lib/ shadows version-pinned ...`. With no snapshot, `cyrius lib sync` could not run
  in CI at all (it fails with `snapshot lib not found at ~/.cyrius/versions/<v>/lib`) and
  the shadow check silently had nothing to compare against. **The stale bundles this cut
  found were therefore invisible to CI by construction, not by oversight.** Both
  workflows now pipe cyrius's own `scripts/install.sh` with the pin from
  `cyrius.cyml`, per patra's convention:

  ```yaml
  CYRIUS_VERSION="$(grep '^cyrius = ' cyrius.cyml | head -1 | sed 's/cyrius = "\(.*\)"/\1/')"
  curl -sSf https://raw.githubusercontent.com/MacCracken/cyrius/main/scripts/install.sh | \
    CYRIUS_VERSION="$CYRIUS_VERSION" sh
  ```

  `release.yml` was already calling an installer, but a third way — downloading the
  tarball and running the `install.sh` *inside it*. It worked; it also meant one repo
  carried two install idioms. Both are now the same step.
- **New CI gate: `Vendored lib/ matches the toolchain pin`.** Runs `cyrius lib sync
  --full` and fails if `git status --porcelain lib/` is non-empty — re-syncing a
  correctly-vendored tree must be a no-op. This is the gate that would have caught this
  cut's drift on the commit that introduced it. `git status --porcelain` rather than
  `git diff --exit-code` because a snapshot that *adds* files (this pin added a whole
  `lib/unicode/` sub-package) shows up as untracked, which `git diff` does not see.
  Verified both ways before shipping: the gate passes on a committed copy of this tree,
  and fails as intended when a stale `lib/fmt.cyr` and a missing `lib/unicode/` file are
  injected. Also verified that the vendored `lib/` is byte-identical to the official
  `cyrius-6.5.35-x86_64-linux.tar.gz` stdlib, so the gate cannot fire spuriously on a
  local-vs-release snapshot difference.
- **New CI gate: `Verify the versioned snapshot exists`** — asserts
  `$HOME/.cyrius/versions/<pin>/lib` is there right after the install step, so a future
  installer regression fails with that sentence rather than as a confusing second-order
  failure in the drift gate.
- **The version-consistency gate accepted any historical CHANGELOG mention.** It ran
  `grep -qE "^## \[${VERSION}\]" CHANGELOG.md`, which passes as long as *some* section
  carries the number — so a release could ship with no entry of its own. Now compares
  against the **top** release entry (skipping `## [Unreleased]`). Confirmed against the
  real file: the old form wrongly passes for `1.1.5`, the new form correctly rejects it.
  Same failure mode, and the same fix, as patra's CI documents.

- **Documentation that had gone factually wrong**, found while re-checking the version
  surface this cut touches:
  - `README.md` — *Status* read **"Pre-1.0 scaffold (0.1.0). Prints version and exits"**,
    six releases after v1.0.0 froze the output shape, and *Shape* still described mihi as
    "currently scaffolded; not yet a published dep". Rewritten: real status, a sample of
    the seven-line output, the `(unknown)` / exit-0 policy, and a build snippet matching
    what CI runs. A speculative "renders via `darshana` ANSI primitives if any color is
    ever desired" bullet was dropped from *Shape* — no such path exists, and *Shape*
    describes what iam is. Colour itself stays a live post-v1 question, not a closed
    one: ADR 0001 §5 keeps the door open for v2.0 behind a new ADR and a default-off
    byte contract.
  - `CLAUDE.md` *Quick Start* — claimed `./build/iam` prints `"iam v0.1.0 — scaffold"`.
    Now also names `cyrius lib sync --full`, which any `[package].cyrius` bump needs and
    which this cut needed.
  - `docs/guides/getting-started.md` — the "Once M1+ ships" hedge (M1–M6 have all
    shipped), plus the omission that mattered most: *Adding a display line* read like a
    5-step checklist when, post-v1.0, adding or reordering a line is a major-version
    event. Says so now.
  - `docs/development/state.md` *Dependencies* — described **mihi 1.1.1** and a live
    `agnosys` dep, neither true since 1.1.3; and *Toolchain* pinned `6.2.22` while the
    manifest said `6.2.37`. Both rewritten against the shipped manifest.
  - `docs/development/roadmap.md` — the *Pending upstream — agnosys → agnodrm* item sat
    unchecked though it was satisfied at 1.1.3. Closed, with the history recorded.
  - `docs/doc-health.md` — every row above claimed ✅ Fresh as of 2026-05-19. Rows for
    the files this cut touched are re-dated with what actually changed; the header now
    says plainly that the untouched rows have *not* been re-read and that a full
    re-sweep is the next doc-health job, rather than leaving the blanket "every row
    reads ✅ Fresh" line standing over demonstrably stale entries.

### Verified

- **archaemenid (Linux 7.1.5):** `./build/iam` renders
  `GPU:    AMD Radeon (PCI 0x1002:0x1638) [3 GiB]`. Linux sysfs
  (`mem_info_vram_total` = 3221225472) and the AGNOS `#89` `vram_mb` field (3072) report
  the same 3 GiB on this silicon, so the line reads identically on both platforms.
- `cyrius build src/main.cyr build/iam` **OK**; `cyrius build --agnos` **OK**;
  `cyrius lint src/display.cyr src/main.cyr` 0 warnings; `cyrius test` **122/0**.
- **The CI changes were exercised locally, not just written.** Both workflow files
  parse as valid YAML with the expected step lists. The pin extraction
  (`grep '^cyrius = '`) returns `6.5.35` against the real manifest. Every gate in
  `build-and-test` and `docs` was replayed step-for-step on this tree and passes. The two
  new gates were checked in both directions: `Vendored lib/ matches the toolchain pin`
  passes against the real committed tree — `cyrius deps && cyrius lib sync --full` leaves
  `git status --porcelain lib/` empty — and fails as intended against an injected stale
  `lib/fmt.cyr` plus a deleted `lib/unicode/categories.cyr`; the hardened
  version-consistency gate rejects `1.1.5` where the old bare grep accepted it. The
  vendored `lib/` was independently diffed against the official
  `cyrius-6.5.35-x86_64-linux.tar.gz` — byte-identical across all 108 stdlib modules,
  which is what makes the drift gate safe to make blocking.
- **Post-refresh gate, whole tree**: `cyrius build` **OK** on all three targets
  (x86_64 / `--agnos` / `--aarch64`), `cyrius lint` **0 warnings** across all four
  `src/*.cyr`, `cyrius test tests/iam.tcyr` **122 passed / 0 failed**. The CI smoke gate
  replayed locally: 7 lines, `Distro Host Kernel Uptime CPU GPU Memory`, exit 0,
  **empty stderr**, and `./build/iam` byte-identical to `./build/iam | cat` (pipe ==
  TTY, ADR 0001 §5).
- **No runtime regression on the login-hot path.** mihi 1.2.3 made `mihi_cpu_model`
  126% slower on purpose (it was reading 3288 of 8192 bytes; now it reads all of them),
  which is the one thing in this span that could have cost iam something. Measured as an
  A/B rather than against the historical figure — old tree and new tree built by the
  same cycc 6.5.35, interleaved, three trials each of the documented N=500 batch on
  archaemenid:

  | | trial 1 | trial 2 | trial 3 | median |
  |---|---:|---:|---:|---:|
  | 1.1.5 pins (mihi 1.2.1 / ai-hwaccel 2.2.6) | 1654 µs | 1628 µs | 1601 µs | **1628 µs** |
  | this tree (mihi 1.2.4 / ai-hwaccel 2.3.18) | 1578 µs | 1601 µs | 1570 µs | **1578 µs** |

  New is marginally *faster*, which is noise rather than a win — the point is that the
  cpu_model change is invisible at iam's scale, because the GPU probe still dominates.
  M5's < 10 ms cold-start gate holds with ~6.3× headroom. Interleaving matters here: the
  historical 1510 µs trend row was taken on a different kernel and a different toolchain,
  so it is not a valid control for this question.

### Notes

- **Line order, label set, label width, `(unknown)` policy, and exit-0 are unchanged** —
  the v1.0 freeze covers those, and this cut moves none of them. Output stays six or seven
  lines. Per `state.md`'s post-v1.0 stewardship clause ("an addition that fits cleanly
  alongside the spine needs ADR justification and re-audit but can ship in a `Minor` cut"),
  this is a `Minor`-eligible change; the version number is left for the maintainer to name.
- **Multi-GPU policy unchanged** — still the first device only (ADR 0002 §3). N GPU lines
  would break the six-or-seven-line count guarantee, which *is* frozen. Consumers needing
  per-device detail call mihi directly.
- **AGNOS rendering is gated on mihi**, not on iam. The GPU row and its suppress-on-zero
  behaviour have existed since v0.4.0; it stays absent on AGNOS until mihi's
  `mihi_gpu_count()` reads syscall #89. No further iam change is required when it lands.
- **`Minor`, not `Breaking`.** The v1.0 freeze covers line order, label set, label
  width, the `(unknown)` policy, and exit-code discipline; this cut moves none of them.
  The GPU value column gains an optional suffix (ADR 0003) and everything else is pins.
- **iam never sees ai-hwaccel's log output, and that is mihi's doing, not luck.**
  `sakshi` defaults to `SK_INFO`, so a bare `registry_detect_no_exec()` writes
  `detect: profiles=N` to the *consumer's* stderr — which for iam would mean junk on the
  terminal above the report. mihi 1.2.2's `_mihi_gpu_ensure()` saves the caller's level,
  clamps to `SK_WARN` for the one detect call, and restores it. Confirmed on this tree:
  stderr is **0 bytes**. Worth knowing because iam is the reason the clamp exists.
- **`[deps.agnosys]` stays gone.** The roadmap's *Pending upstream* item ("drop the
  transitive agnosys dep") was already satisfied at 1.1.3 when mihi rewired to
  `sys_uname` / `sys_sysinfo`; this cut removes the last physical trace, the orphaned
  `lib/agnosys-core.cyr`. Item closed.
- **One pre-existing `cyrius fmt --check` divergence is left standing**, at
  `src/main.cyr:109-110` and `:135`: three continuation lines, across two call sites, use
  aligned-to-open-paren indentation where canonical style is 2 spaces per open paren. 6.5.35's formatter flags
  what 6.2.37's accepted. Deliberately not auto-formatted — iam's CI gates on `cyrius
  lint` (clean) and not on `cyrius fmt`, and the rewrite trades readable argument
  alignment for the canonical indent. mihi took the opposite call for its test file at
  1.2.2; iam's is a maintainer decision, not a blocker.

## [1.1.5] — 2026-07-02

**Fix: iam faulted on agnos (never rendered) — a user-stack overflow, plus the CPU line.**
On agnos, `run /bin/iam` died with a CPL3 page fault (write to the unmapped page just
below `rsp`) before printing anything. Root cause: `main` put ~17.6 KB of scratch buffers
on the stack, but agnos gives a program only ~12 KB of usable user stack (`elf.cyr` exec
layout), so the frame overflowed. (Found via `-d int`: `v=0e e=0006 cpl=3 CR2=rsp-8`.)

### Changed

- **Heap-allocate the two large `/proc` scratch buffers** (`cpubuf` 8 KiB + `membuf`
  4 KiB) instead of stack `var[N]` (`src/main.cyr`) — they're only used on Linux; agnos
  fills CPU from CPUID and memory from `sysinfo`#35. Drops the `main` frame from ~17.6 KB
  to ~5.5 KB, well under the ~12 KB agnos user-stack budget.
- **Bumped `[deps.mihi]` `1.2.0` → `1.2.1`** — picks up the fix that actually compiles the
  CPUID brand-string asm into the agnos build (1.2.0 had it `#ifdef`'d out on `--agnos`).

### Verified

- **agnos (QEMU/KVM):** iam renders the full system card with the real CPU brand
  (`AMD Ryzen 7 5800H with Radeon Graphics`) — `agnos/scripts/iam-agnos-verify.py` **PASS**,
  no fault. Native + `--agnos` builds green; test 105/0.

### Notes

- Surfaced (not fixed here): **~12 KB of usable user stack is a real agnos kernel limit** —
  iam is just the first program to hit it. A kernel fix (start `rsp` near the top of the
  mapped 2 MB stack page) would give every program a real stack.

## [1.1.4] — 2026-07-02

### Changed

- **Bumped `[deps.mihi]` `1.1.3` → `1.2.0`** — picks up mihi's sovereign CPUID CPU-model
  probe. On **AGNOS** the `CPU:` line now renders the real processor brand (via CPUID
  leaves 0x80000002/3/4) instead of `(unknown)` — agnos has no `/proc/cpuinfo`, which the
  old mihi path read. The `cpus` count now reflects the kernel's enumerated CPUs
  (`sysinfo`#35) rather than a stale hardcoded `1`. No iam-side code or API change — the
  fix rides entirely in the mihi bundle. Linux output unchanged. Rebuilt: native shows
  the real brand (`AMD Ryzen 7 5800H …`); `iam --agnos` compiles against the tagged bundle.

## [1.1.3] — 2026-06-22

### Changed

- **Dropped the `agnosys` git dep; rewired onto the native `sys` stdlib module.** cyrius
  retired the stale stdlib `agnosys` snapshot at **6.2.37**; iam carried `[deps.agnosys]`
  (1.4.0) only because mihi's bundle read uname/sysinfo through `agnosys_uname`. With **mihi
  `1.1.2` → `1.1.3`** rewired to `sys_uname` / `sys_sysinfo` (`lib/sys.cyr`), iam drops the
  agnosys git dependency, adds `"sys"` to the stdlib, bumps cyrius `6.2.22` → `6.2.37`, and
  adds a `cyrius lib sync` CI step (the `sys` module is opt-in-vendored). **Host + `--agnos`
  builds verified clean** — iam still renders its system card on agnos via mihi's probes. No
  display-surface change.

## [1.1.2] — 2026-06-19 (AGNOS verified)

### Changed

- **`[deps.mihi] tag` 1.1.1 → 1.1.2** — mihi gained the AGNOS build-target probe branches (sysinfo#35 RAM/uptime, single-core CPU count, `AGNOS` distro). iam itself needs **no source change**: it is pure presentation — every line routes through a `mihi_*` probe, so mihi's agnos branches carry it across.

### Verified

- **Renders the full system card on real agnos (kernel 1.45.10) under QEMU** — `Distro: AGNOS` / `Kernel: AGNOS` / `Uptime` / `CPU` / `Memory: 128 MiB`, driven through agnsh. Harness: `agnos/scripts/iam-agnos-verify.py`.

## [1.1.1] — 2026-06-18 (toolchain + dep refresh)

### Changed

- **`[package].cyrius` 6.0.1 → 6.2.22.** Adopts the 6.2.x stdlib
  reorg (below); matches mihi 1.1.1's own toolchain pin.
- **`[deps.mihi] tag` 1.0.0 → 1.1.1.** mihi's identity probes now
  read `uname` / `sysinfo` through `agnosys_uname`, so the bundle
  (`dist/mihi.cyr`) references agnosys symbols — see the new dep
  below. `[deps.ai-hwaccel]` stays at **2.2.6** (mihi 1.1.1 pins the
  same transitive; the 2.3.x line is not what mihi references).
- **stdlib reorg (tracks mihi 1.1.1's `cyrius.cyml`):**
  - `agnosys` left the stdlib list and is now a proper git
    dependency — `[deps.agnosys] tag = "1.4.0"`,
    `modules = ["dist/agnosys-core.cyr"]` (the AGNOS-portable `core`
    bundle; pin matches mihi 1.1.1's transitive). dist/mihi.cyr
    needs the agnosys symbols present at parse time.
  - `json` → `bayan` in the stdlib list. The standalone `json`
    module was carved into the bundled `bayan` distribution and
    folded back byte-identical via sandhi; mihi's `registry_to_json`
    symbols resolve through bayan's back-compat aliases. `cyrius lib
    sync` re-vendored the 6.2.22 stdlib snapshot into `lib/`.

**No source changes.** No `src/*.cyr` edits; runtime output is
byte-for-byte identical to v1.1.0 on archaemenid (Distro / Host /
Kernel / Uptime / CPU / GPU / Memory spine unchanged). Build, lint,
and tests pass; `cyrius.lock` regenerated (110 deps locked).

## [1.1.0] — 2026-06-06 (cycle-open: AGNOS as a build target)

### Added

- **AGNOS platform support — cycle opened** (VERSION → 1.1.0). An AGNOS-target build so `iam` renders the system-info splash natively on AGNOS, displaying the fields mihi now reads from the `uname`#34 / `sysinfo`#35 kernel syscalls; the GPU/distro lines suppress where AGNOS has no source yet. Inline; no platform-abstraction layer yet.

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
