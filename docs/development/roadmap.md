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

### M5 — Harden + dogfood (v0.9.0)

- Maintainer uses `iam` in MOTD for one release cycle
- Invocation-time benchmark: `iam` cold-start < 10ms on archaemenid
- 3-point benchmark trend in `docs/benchmarks.md`
- P(-1) hardening pass complete — security audit doc filed

### M6 — v1.0.0

- Pin to mihi 1.0.x (mihi must ship 1.0 first; the dep gate matters)
- Output shape frozen (the M4 ADR becomes contract)
- CHANGELOG `Breaking` section for the freeze
- v1.0.0 cut

## Out of scope (for v1.0)

The list keeps future contributors from adding to v1.0 by accident.
*Most of these are out of scope **forever**, not just for v1.0* —
they're load-bearing non-features.

- **Theming engine** — never. The output shape is the contract.
- **Plugin system** — never.
- **Configurable output (env vars, dotfiles, flags)** — never.
- **ASCII logos** — never. Render banners via `BannerManor`.
- **`--json` / `--csv` output formats** — never. If you need machine-
  readable system info, consume `mihi` directly; don't shoehorn it
  into `iam`'s display surface.
- **Color** — out of scope for v1.0; reconsider for v2.0 if there's
  user demand. Default plain stays a v1.0 gate.
- **Caching** — every invocation is a fresh mihi probe. No cache.
- **Network ANYTHING** — never.

## Cross-references

- [`state.md`](state.md) — live status (mihi version pin, output lines)
- [`../../CHANGELOG.md`](../../CHANGELOG.md) — release history
- [mihi roadmap](https://github.com/MacCracken/mihi/blob/main/docs/development/roadmap.md) — the dep that gates every iam milestone
