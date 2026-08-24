---
name: iam Documentation Health
description: Living state of doc currency in the iam repo — fresh / stale / evergreen / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — iam

> **Last refresh**: 2026-08-23 (v1.1.6 cut — GPU-memory suffix (ADR 0003) + toolchain/dep-floor refresh: cyrius `6.2.37` → `6.5.35`, mihi `1.2.1` → `1.2.4`, ai-hwaccel `2.2.6` → `2.3.18`, `sakshi` added to the stdlib list, `lib/` re-vendored and ten orphans pruned). Rows touched this cut: `VERSION`, `cyrius.cyml`, `cyrius.lock`, `CHANGELOG.md`, `docs/benchmarks.md`, `development/state.md`, `development/roadmap.md`. **Note**: rows *not* listed there carry dates from the v0.9.0-era sweep and have not been re-read since — the row dates are honest, the blanket "every row reads ✅ Fresh" claim below is the stale part, and a full re-sweep is the next doc-health job. **Prior refresh**: 2026-05-19 v0.9.0 RC — F-001 sanitizer landed, follow-up audit filed. **Prior**: 2026-05-19 v0.8.0 — ADR 0002 accepted, output-shape reorder. | **Refresh cadence**: when docs are touched, update the affected row.
> **Scope**: This repo only (`iam`) — the entire `docs/` tree plus root-level files (README, CHANGELOG, CLAUDE.md, VERSION, cyrius.cyml, cyrius.lock, SECURITY.md, CONTRIBUTING.md, CODE_OF_CONDUCT.md, LICENSE). Mihi's docs live in the mihi repo and are not audited here; cross-repo dep state is captured in [`development/state.md`](development/state.md), not here.
>
> **Convention adopted from cyrius / agnosticos** (2026-05-19): same tier-table shape, scaled to iam's much smaller tree (~14 markdown docs vs cyrius's ~105). One ledger row per doc; rewrite-in-place as docs change.

This is a **ledger**, not a one-time audit. Rewrite-in-place as docs change.

---

## At a glance — 2026-05-19 inventory (v0.9.0 RC cut)

**15 markdown docs** under `docs/` (+1 since v0.8.0: the v0.9.0
audit doc) + 9 root-level files (README, CHANGELOG, CLAUDE.md,
SECURITY.md, CONTRIBUTING.md, CODE_OF_CONDUCT.md, VERSION,
cyrius.cyml, cyrius.lock). Bucket counts:

| Bucket | Count | What it means |
|---|---|---|
| ✅ **Fresh / touched in current cycle** | 20 | All operational + root docs touched across the v0.5.0 → v0.9.0 arc (state.md / roadmap.md / CHANGELOG / VERSION / three audit docs / benchmarks.md / ADR 0001 (Superseded) / ADR 0002 (Accepted) / sample-output.txt / README / CLAUDE.md / SECURITY.md / etc.). |
| 🟡 **Stale — refresh in place** | 0 | None flagged. |
| 🟠 **Read-through outstanding** | 0 | None flagged. |
| 🔵 **Probably evergreen** | 4 | `docs/adr/template.md`, `docs/adr/README.md`, `docs/architecture/README.md`, `CODE_OF_CONDUCT.md` — convention / framing docs; re-read pass at major releases, not weekly. |
| 📦 **Archive — frozen by design** | 0 | No `docs/development/archive/` directory yet; audit docs are date-stamped artifacts (not in-place refreshed) but live in their `docs/audit/` tier, not an archive. ADR 0001 stays in `docs/adr/` with `Superseded` status — same archive-in-place pattern. |
| ❓ **Open strategic question** | 0 | None outstanding. F-001 mitigation closed at v0.9.0; mihi 1.0 ship is an external dependency, not a strategic question. |

**Why now**: iam's doc tree has been actively maintained since scaffold (every release cuts a state.md refresh and a CHANGELOG entry per CLAUDE.md), but the *aggregate* currency had no surface. This file is that surface — same convention cyrius, agnosticos, and the rest of the first-party set use.

**2026-05-19 sweep notes (v0.9.0 RC cut)**: F-001 sanitizer landed in this cut — TTY-escape mitigation at the renderer boundary (`iam_copy_value` in `src/display.cyr`). Doc-side effect: new audit doc `2026-05-19-v0.9.0-audit.md` filed superseding M5.5 for the v0.8.0 reorder + v0.9.0 F-001 scope; `docs/benchmarks.md` gains a fourth trend row (1510 µs median, v0.9.0 stays inside the v0.5.0 noise floor) and bumps the toolchain row 6.0.0 → 6.0.1 (resolves the drift the v0.7.0 scaffold flagged); state.md *Version* / *Audit* / *Tests* / *Benchmarks* / *Next* sections refreshed; roadmap M6 marks v0.9.0 RC shipped, mihi 1.0 ship now the only remaining v1.0 gate. **No remaining ❓ Open strategic questions; no v1.0 blockers on the iam side.**

**Prior sweep (2026-05-19, v0.8.0)**: ADR 0002 accepted — output-shape reorder to identity → runtime → hardware spine. Closes the lone ❓ that the v0.7.0 scaffold surfaced.

**Prior sweep (2026-05-19, v0.7.0)**: initial scaffold of this file. No drift to retire — every doc was touched within the active 0.5.0 → 0.7.0 milestone-closure arc.

---

## Tier 1 — Structural docs (root + `docs/` root)

| File | Last touched | Status | Action |
|---|---|---|---|
| `README.md` | 2026-08-23 | ✅ Fresh | Top-level project README. **Was the worst drift in the tree** — the *Status* section still read "Pre-1.0 scaffold (0.1.0). Prints version and exits" six releases after v1.0.0 froze the output shape, and the *Shape* section still called mihi "scaffolded; not yet a published dep." Rewritten at v1.1.6: real status, a sample of the seven-line output, the `(unknown)`/exit-0 policy, and a build snippet that matches what CI actually runs. Prior row claimed ✅ Fresh since 2026-05-19, which is exactly the failure mode this ledger exists to catch. |
| `CHANGELOG.md` | 2026-08-23 | ✅ Fresh | **Source of truth per CLAUDE.md.** Through v1.1.6. Refreshed every release. |
| `CLAUDE.md` | 2026-08-23 | ✅ Fresh | Process + procedures + project-identity. Volatile state delegated to `docs/development/state.md` per its own principle. *Quick Start* corrected at v1.1.6 — it still claimed `./build/iam` prints `"iam v0.1.0 — scaffold"`, and now names `cyrius lib sync --full` (needed after any `[package].cyrius` bump) and the explicit `cyrius test tests/iam.tcyr` form CI uses. |
| `VERSION` | 2026-08-23 | ✅ Fresh | Single source of truth for version (`1.1.6` at last edit). `cyrius.cyml [package].version = "${file:VERSION}"` auto-tracks. |
| `cyrius.cyml` | 2026-08-23 | ✅ Fresh | Manifest. Toolchain pin `6.5.35`; mihi `1.2.4`; ai-hwaccel `2.3.18`; 21-module stdlib list, byte-equal to mihi's `dist/mihi.deps` sidecar. |
| `cyrius.lock` | 2026-08-23 | ✅ Fresh | Generated artifact. Refreshed by `cyrius deps` whenever pins move. 110 hash entries + 2 commit pins at v1.1.6. |
| `SECURITY.md` | 2026-05-19 | ✅ Fresh | Threat-surface summary + reporting address. Threat model matches the M5 / M5.5 audit docs (mihi return-data abuse + TTY escape injection). |
| `CONTRIBUTING.md` | 2026-05-19 | 🔵 Evergreen | Contribution conventions; touched only when the process changes. |
| `CODE_OF_CONDUCT.md` | 2026-05-19 | 🔵 Evergreen | Standard COC; rarely changes. |
| `LICENSE` | 2026-05-19 | 🔵 Evergreen | GPL-3.0-only. Frozen by the license choice. |
| `docs/benchmarks.md` | 2026-08-23 | ✅ Fresh | Invocation-time benchmark methodology + four-point trend (v0.3.0 → M3@9df0859 → v0.5.0 → v0.9.0), plus a v1.1.6 interleaved A/B section kept deliberately *off* that table (host kernel and compiler both moved, so a fifth row would not be comparable). M5 < 10 ms gate holds — 1628 µs → 1578 µs median across the 1.1.5 → 1.1.6 dep floor, i.e. mihi 1.2.3's 126%-slower `mihi_cpu_model` is invisible at iam's scale. **Stale sub-part**: the *Reference host* table still records kernel 7.0.5 / cyrius 6.0.1, which is correct for the four-point trend and wrong for the A/B; the A/B section states its own host state inline. Refresh when a new benchmark run lands or a perf-relevant change ships. |

---

## Tier 2 — Architecture notes (`docs/architecture/`)

Non-obvious constraints not derivable from the code. iam's surface is small enough that no concrete entries exist yet.

| File | Last touched | Status | Action |
|---|---|---|---|
| `README.md` | 2026-05-19 | 🔵 Evergreen | Framing doc explaining what goes here vs ADRs vs guides. No numbered entries yet — by design (the code is small enough to read). Touch the first time a non-obvious invariant earns a numbered entry. |

---

## Tier 3 — Operational / Development (`docs/development/`)

> **Important framing**: `state.md` + `roadmap.md` form the **canonical operational surface**. CLAUDE.md delegates volatile state to `state.md`, and `roadmap.md` is the milestone-pinning artifact. Both rotate every release.

| File | Last touched | Status | Action |
|---|---|---|---|
| `state.md` | 2026-08-23 | ✅ Fresh | **Rotates every release.** v1.1.6 — *Version* rewritten (GPU-memory suffix + dep-floor refresh), *Toolchain* pin corrected `6.2.22` → `6.5.35` (had drifted two cuts behind the manifest), *Dependencies* rewritten from scratch (it still described mihi 1.1.1 and a live `agnosys` dep, both wrong since 1.1.3) and given a vendored-`lib/` subsection, *Benchmarks* given a v1.1.6 row with a non-comparability note. |
| `roadmap.md` | 2026-08-23 | ✅ Fresh | **Rotates per milestone.** M0 through M6 shipped (v1.0.0 froze the output shape). *Pending upstream — agnosys → agnodrm* checked off at v1.1.6: the dep itself came out at 1.1.3 when mihi rewired to `sys_uname`, and the orphaned `lib/agnosys-core.cyr` was pruned at 1.1.6, so the item had been silently satisfied for three cuts. "Out of scope (for v1.0)" section restructured into honest "Not iam's job" vs "Deferred" splits at v0.6.0. |

---

## Tier 4 — ADRs (`docs/adr/`)

2 ADRs, 1 accepted + 1 proposed. ADRs document decisions; re-read pass at major releases.

| File | Last touched | Status | Notes |
|---|---|---|---|
| `README.md` | 2026-05-19 | 🔵 Evergreen | Convention doc — filename rules, status lifecycle, ADR-vs-architecture-vs-guide table. |
| `template.md` | 2026-05-19 | 🔵 Evergreen | Skeleton for new ADRs. Touch when the convention itself changes. |
| `0001-output-shape.md` | 2026-05-19 | 🔵 Frozen artifact (Superseded by 0002 at v0.8.0) | Locks label column, `(unknown)` fallback, no-color / pipe-equivalence, exit-0 contract. §1, §4, §5, §6 still canonical; §2 (line order) and §3 (optional-line policy) superseded by ADR 0002. Body left intact as historical record per ADR convention. |
| `0002-output-shape-reorder.md` | 2026-05-19 | ✅ Fresh (Accepted at v0.8.0) | Identity → runtime → hardware spine (Distro → Host → Kernel → Uptime → CPU → GPU? → Memory). Supersedes ADR 0001 §2-§3 only; §1, §4, §5, §6 of ADR 0001 carry over. **This is the v1.0 byte contract.** |

---

## Tier 5 — Audits (`docs/audit/`)

Periodic audit reports; per-audit timestamped (don't refresh in place — supersede with a new audit doc per the audit-trail convention).

| File | Last touched | Status |
|---|---|---|
| `2026-05-19-audit.md` | 2026-05-19 | 🔵 Dated artifact — M5 P(-1) audit (v0.5.0 scope). Superseded; kept in-place as historical record. |
| `2026-05-19-m5.5-audit.md` | 2026-05-19 | 🔵 Dated artifact — M5.5 refreshed audit (v0.6.0/v0.7.0 scope). Superseded by the v0.9.0 doc; kept in-place. |
| `2026-05-19-v0.9.0-audit.md` | 2026-05-19 | ✅ Fresh — v0.9.0 RC follow-up audit. **Verdict pass; F-001 RESOLVED; F-002 carries INFO; no remaining v1.0 blockers on the iam side.** Section A reviews `iam_copy_value` design + call-site fan-in + 15 byte-exact tests; Section B re-walks src/ against the M5/M5.5 checklist with v0.9.0 line numbers; conclusion names mihi 1.0 as the only external v1.0 gate. |

Per CLAUDE.md *Process P(-1)*: next renewal trigger is the mihi 1.0 repin (mandatory dep major bump), or any new source change in `src/*.cyr` beyond docstring/comment edits, or the M6 v1.0 cut — whichever comes first.

---

## Tier 6 — Guides + Examples (`docs/guides/`, `docs/examples/`)

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-08-23 | ✅ Fresh | Build / layout / "adding a display line" walk-through. The flagged "Once M1+ ships" hedge is **resolved** at v1.1.6 — the layout section now describes the shipped source tree (including vendored `lib/`), the build snippet matches CI, and *Adding a display line* now states the thing it most needed to: since v1.0.0 froze the shape, adding or reordering a line is a major-version event, not an `Added` entry. |
| `examples/sample-output.txt` | 2026-05-19 | ✅ Fresh | Canonical 7-line output captured on archaemenid. Referenced from ADR 0001. Refresh whenever the output shape changes (which won't happen post-v1.0 freeze). |
| `examples/.gitkeep` | — | 📦 Trivial | Directory anchor. Not tracked here further. |

---

## Refresh procedure

When docs are touched:

1. Find the affected row in the relevant tier table.
2. Update **Last touched** to the new date.
3. Update **Status** if the bucket changed.
4. Update **Action** if the next step changed.
5. If a doc moved or was archived, update its row.
6. Re-anchor "Last refresh" date in the header.

When the bucket counts at the top drift by more than ~2 in any cell, refresh the at-a-glance table.

This file's refresh cadence is **opportunistic** (touched when other docs are touched), not periodic.

---

## What this file is NOT

- Not a substitute for [`development/state.md`](development/state.md) (which holds current-cycle version / audit / next-step state).
- Not a CHANGELOG (which records what shipped, not what's stale).
- Not a TODO list (open work for the project lives in [`development/roadmap.md`](development/roadmap.md)).
- Not a per-doc review log (this is the ledger of where each doc stands, not the per-doc reasoning).

---

## Forward doc-policy commitments

Items that are *scheduled* doc decisions, not stale state. Surfaced here so they aren't forgotten when the trigger date arrives.

| # | Commitment | Trigger | Source | Notes |
|---|---|---|---|---|
| 1 | **Per-release state refresh** — `docs/development/state.md` Version / Audit / Next sections refreshed at every release cut per the file's own *"Refreshed every release"* contract. | Every release | [`development/state.md`](development/state.md) header | Manual, alongside `VERSION` + CHANGELOG bump. |
| 2 | **Periodic audit** — full P(-1) source review + dep-tree CVE sweep before the M6 v1.0 freeze (and on any mihi / ai-hwaccel / cyrius major bump). | Before v1.0; on dep-major bump | [`CLAUDE.md`](../CLAUDE.md) *Process P(-1)* | Most recent: 2026-05-19 v0.9.0 audit (M5 → M5.5 → v0.9.0 chain). Next trigger: mihi 1.0 repin (mandatory) or any new `src/*.cyr` change — see the v0.9.0 audit's *Next audit trigger* clause. |
| 3 | **F-001 mitigation** — **CLOSED at v0.9.0**. `iam_copy_value` sanitizer landed in `src/display.cyr`, wired into all three renderers, 15 byte-exact tests cover the surface, bench confirms zero measurable cost. Row kept one cycle as historical commitment. | (resolved) | [`audit/2026-05-19-v0.9.0-audit.md`](audit/2026-05-19-v0.9.0-audit.md) §A | Closed. |
| 4 | **ADR 0002 resolution** — **CLOSED at v0.8.0 (Accepted)**. Output-shape reorder landed; ADR 0001 §2-§3 superseded; tests regenerated; sample-output regenerated. Row removable at next sweep if uneventful. | (resolved) | [`adr/0002-output-shape-reorder.md`](adr/0002-output-shape-reorder.md) | Closed. |
| 5 | **Benchmark refresh** — re-run the bench harness against current `build/iam` whenever a perf-relevant change ships. | On perf-relevant change | [`benchmarks.md`](benchmarks.md) | Current trend: 4 points (v0.3.0 / M3@9df0859 / v0.5.0 / v0.9.0). Toolchain pin drift resolved 6.0.0 → 6.0.1 at the v0.9.0 refresh. Next likely refresh: post-mihi-1.0 repin. |

---

*Initial scaffold: 2026-05-19 (v0.7.0 — M5.5 audit closeout). Convention pattern matches `cyrius/docs/doc-health.md` (and the broader first-party set: agnosticos / vidya / phylax / etc.), scaled to iam's ~14-doc tree. Refresh in place when docs are touched.*
