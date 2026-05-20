# iam — Roadmap

> Milestone plan through v1.0. State lives in [`state.md`](state.md);
> this file is the sequencing — what ships, in what order, against
> what dependency gates.

## v1.0 criteria

The iam v1.0 contract: a stable, terse, whoami-simple output that
shells can rely on for MOTD and screenshot flex. Frozen output
shape; mihi 1.0 dep pinned.

- [ ] Output shape frozen — exact line order, label format, exit codes
- [ ] Pinned to [`mihi`](https://github.com/MacCracken/mihi) 1.0.x
      (consumed for every system-fact line)
- [ ] Test coverage: happy path per output line + mihi-error
      degraded-output path; 50+ assertions (iam is small)
- [ ] Benchmarks captured in `docs/benchmarks.md` — `iam` invocation
      time is a login-path concern (target: < 10ms cold start)
- [ ] CHANGELOG complete from v0.1.0 onward
- [ ] Security audit pass (`docs/audit/YYYY-MM-DD-audit.md`)
- [ ] No theming engine, no plugin system, no configurable output —
      this is a v1.0 gate, not just a non-goal

## Milestones

### M0 — Scaffold (v0.1.0) — ✅ shipped 2026-05-19

- `cyrius init` scaffold landed
- `./build/iam` prints scaffold version and exits
- Doc-tree per [first-party-documentation.md](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/first-party-documentation.md)

### M1+M2 — Pin mihi + six display lines (v0.3.0) — ✅ shipped 2026-05-19

Collapsed into one cut: mihi was already at 0.7.0 (past M3's gate)
by the time iam reached M1, so the M1 (CPU/RAM/kernel) and M2
(host/distro/uptime) line sets shipped together.

- `[deps.mihi]` pinned to 0.7.0; `[deps.ai-hwaccel]` 2.2.6 pulled in
  transitively (mihi/gpu.cyr references inside `dist/mihi.cyr`).
- Display lines: `CPU`, `Memory`, `Kernel`, `Host`, `Distro`, `Uptime`.
- Pipe-aware: identical output on TTY and pipe (no color emitted).
- Tests cover formatter logic (51 assertions); end-to-end probe
  wiring verified by running `./build/iam` on the maintainer's box.

### M3 — GPU line (v0.4.0) — ✅ shipped 2026-05-19

- Trailing `GPU:` line emitted when `mihi_gpu_count() > 0`,
  suppressed entirely otherwise (zero accelerators is not an error).
- Multi-GPU policy (recorded for M4's ADR): show the first device.
  Whoami-simple wins over multi-line surface; consumers needing
  per-device detail call mihi directly.
- CI smoke gate accepts 6 or 7 lines; DCE parity check
  cross-validates GPU presence between non-DCE and DCE'd builds.
- **Dep gate**: mihi ≥ 0.4.0 (satisfied — mihi was at 0.7.0).
- **Acceptance**: `iam` on archaemenid (AMD Ryzen 7 5800H, integrated
  Radeon) prints "GPU: AMD Radeon (PCI 0x1002:0x1638)".

### M4 — Output-shape ADR (v0.5.0) — ✅ shipped 2026-05-19

All three M4 deliverables in one cut: ADR + sample output + the
byte-exact test harness (achieved via an `iam_render*` refactor —
renderers now write into a caller-buffer instead of issuing per-
line write syscalls, which made the contract observable to tests).

- ✅ ADR: `docs/adr/0001-output-shape.md` — line order, 8-byte label
  column, `(unknown)` fallback, single-optional-GPU rule, no-color
  / pipe-equivalence, exit-0 contract, alternatives-considered.
- ✅ Sample output captured in `docs/examples/sample-output.txt`.
- ✅ 39 byte-exact tests covering per-label padding, fallback paths,
  six-/seven-line assembled output, all-probes-failed degraded.
- **Dep gate**: none (decision is local).
- **Acceptance**: ADR landed; sample output reproducible across
  runs; test suite executes the contract on synthetic inputs.

### M5 — Harden + dogfood (v0.6.0) — ✅ shipped 2026-05-19

All four deliverables landed. v0.6.0 cuts on M5 completion; the
original "v0.9.0 = M5 baseline" plan was telescoped — M5.5
(security re-audit, v0.7.0) and the v0.9.0 RC now sit between M5
and M6.

- ✅ Invocation-time benchmark: `iam` cold-start ~1.5 ms on
      archaemenid (M5 gate: < 10 ms). 6.5× headroom — see
      `docs/benchmarks.md`.
- ✅ 3-point benchmark trend in `docs/benchmarks.md` —
      v0.3.0 (~555 µs, pre-GPU) → M3@9df0859 (~1514 µs, +GPU
      probe) → v0.5.0 (~1511 µs, +single-flush). Trend caught the
      GPU probe as the dominant cost; M4 single-flush refactor
      was nominal at this scale (recorded honestly).
- ✅ P(-1) hardening pass — security audit filed at
      `docs/audit/2026-05-19-audit.md`. One open finding (F-001:
      TTY-escape sanitization on mihi strings, LOW) to address
      before M6 freeze; one INFO note (F-002: strlen invariant
      cross-referenced to mihi audit).
- ✅ Maintainer uses `iam` in MOTD for one release cycle —
      dogfood ran from 2026-05-19 v0.5.0 cut through the v0.6.0
      cut (`~/.local/bin/iam` → `build/iam`, called from
      `~/.zshrc`, every interactive shell). No issues observed
      across the uptime cycle / multi-tab spawn pattern. The
      cycle-marker convention is "next release cuts" → v0.6.0
      IS the marker.

### M5.5 — Security + code re-audit (v0.7.0) — ✅ shipped 2026-05-19

All three M5.5 deliverables landed in one cut. Source `src/*.cyr`
unchanged from v0.5.0 / v0.6.0; the v0.7.0 release is pure
audit-trail closure.

- ✅ **Web research**: 0day / CVE check against the dep tree —
      mihi @ 0.7.0, ai-hwaccel @ 2.2.6, cyrius 6.0.1 toolchain,
      stdlib bundle. **Clean**: no public CVE / advisory entries
      against any direct dep (all first-party MacCracken
      projects). Name-collision noise ("Cyrus IMAP" / "Cyrus SASL"
      for cyrius; "NVIDIA Container Toolkit" CVEs for ai-hwaccel)
      documented and dismissed. Queries + sources captured in the
      refreshed audit doc.
- ✅ **Code re-review**: full re-walk of `src/*.cyr` confirmed
      zero drift since the v0.5.0 audit baseline
      (`git diff a57c17b..HEAD -- src/` empty). F-001 confirmed
      still open, F-002 confirmed still INFO. Build / lint /
      test / runtime smoke re-verified against v0.6.0 codebase.
- ✅ **Refreshed audit doc**: `docs/audit/2026-05-19-m5.5-audit.md`
      filed. Supersedes the 2026-05-19 M5 audit for the M5.5
      scope; the M5 doc stays in the directory as historical
      record per audit-trail convention. Conclusion section names
      F-001 as the lone v1.0 blocker and F-002 as accepted risk.
      The `-m5.5-` infix disambiguates the same-day collision with
      the M5 audit filename.
- **Dep gate**: none (audit was local + read-only).
- **Acceptance**: refreshed audit doc filed; CVE search returned
      clean across the dep tree; no new findings escalated to v1.0
      blockers.

### M6 — v0.9.0 RC + v1.0.0

- **v0.9.0 RC** cuts when F-001 mitigation (TTY-escape sanitization
  at the renderer boundary) has landed and a follow-up audit
  confirms the sanitizer's bounds. M5.5 cleared the audit-side
  prerequisite at v0.7.0. Last pre-freeze release; sits as the
  release candidate while we wait for mihi 1.0.
- Pin to mihi 1.0.x (mihi must ship 1.0 first; the dep gate matters)
- Output shape frozen — ADR 0002 becomes the v1.0 contract
  (accepted at v0.8.0; identity → runtime → hardware spine). ADR 0001
  carries through for §1, §4, §5, §6.
- CHANGELOG `Breaking` section for the freeze (post-freeze, output-
  shape changes leave the pre-v1.0 grace zone for good)
- v1.0.0 cut

## Out of scope (for v1.0)

The list keeps future contributors from adding to v1.0 by accident.
Split into two honest categories: items that would make iam stop
being iam (the load-bearing non-features), and items that are
deferred for sequencing reasons and may land in a later cut.

### Not iam's job (use the right tool)

These would change iam's identity. Adding them turns iam into
something else (fastfetch, neofetch, a config dumper, a JSON probe).
If you want one of these, the linked tool already exists.

- **Theming *engine*** (iam shipping theme files, palette variants,
  per-line color rules, swappable layouts) — not iam's job. That's
  what neofetch became. iam's contract is "say what the system
  is," not "look pretty in seventeen ways." *Consuming* shell
  theming (terminal palette, `NO_COLOR`, prompt-toolkit
  conventions) is a different conversation and lives in the
  *Deferred* list below under Color / shell-theme integration.
- **Plugin system** — iam is a thin presentation layer over mihi.
  Pluggability lives in mihi (where probes are added) or in a
  sibling tool that consumes mihi.
- **Configurable output (env vars, dotfiles, flags)** — the byte
  contract is the API. Consumers who want a different shape pipe
  iam through `awk` / `cut` / `jq` like they would `whoami`.
- **ASCII logos** (distro art, banners, decoration) — iam stays
  whoami-simple. If you want a banner, `BannerManor` is the
  sibling tool for that. If you want a distro logo next to your
  system info, `neofetch` / `fastfetch` already exist and do
  that well. If you want iam-but-with-a-logo, the GPL-3.0
  license enables a fork — that's the right release valve, not
  growing iam's surface.
- **`--json` / `--csv` output formats** — consume `mihi` directly
  for machine-readable system info. iam's display surface stays
  one-fact-per-line plain text.
- **Caching** — every invocation is a fresh mihi probe. A cache
  layer would mean iam owns staleness semantics; that complexity
  belongs in mihi if anywhere.

### Deferred (may reopen later)

These are sequencing decisions, not identity constraints. The
gating reason is named so a future reader knows when the
conversation reopens.

- **Color / shell-theme consumption** — deferred to v2.0
  conversation. The intent is *consuming* the user's existing
  shell theming (terminal palette via standard escape codes,
  `NO_COLOR` / `FORCE_COLOR` env conventions, prompt-toolkit
  semantics) — not iam shipping its own themes. Default-plain
  stays a v1.0 gate so the byte contract freezes against the
  no-color baseline; color additions post-v1.0 land as opt-in
  and preserve plain bytes as the default pipe-target.
- **Network probes** (e.g. local IP, link state) — deferred until
  kernel-side bring-up is operational. Adds a real probe surface
  to mihi first; iam adopts after.
- **Runtime state** (Battery %, Swap used, Disk used%) — post-v1.0
  consideration. The v1.0 freeze should ship with the stable
  identity → kernel → uptime → hardware set; runtime-state lines
  are the natural v2.0 conversation, additive on top of a frozen
  base.

## Cross-references

- [`state.md`](state.md) — live status (mihi version pin, output lines)
- [`../../CHANGELOG.md`](../../CHANGELOG.md) — release history
- [mihi roadmap](https://github.com/MacCracken/mihi/blob/main/docs/development/roadmap.md) — the dep that gates every iam milestone
