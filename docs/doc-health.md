---
name: iam Documentation Health
description: Living state of doc currency in the iam repo — fresh / stale / evergreen / open-question, refreshed as docs are touched
type: state
---

# Documentation Health — iam

> **Last refresh**: 2026-05-19 (v0.8.0 cut — ADR 0002 accepted, output-shape reordered to identity → runtime → hardware spine; ❓ Open strategic question closed). Tree is 14 markdown docs + the canonical root files; every row currently reads ✅ Fresh because the v0.5.0 → v0.6.0 → v0.7.0 → v0.8.0 cuts touched every doc that mentions version, audit, output shape, or roadmap state. No open strategic questions outstanding. **Prior refresh**: 2026-05-19 v0.7.0 — initial scaffold, M5.5 audit closeout. | **Refresh cadence**: when docs are touched, update the affected row.
> **Scope**: This repo only (`iam`) — the entire `docs/` tree plus root-level files (README, CHANGELOG, CLAUDE.md, VERSION, cyrius.cyml, cyrius.lock, SECURITY.md, CONTRIBUTING.md, CODE_OF_CONDUCT.md, LICENSE). Mihi's docs live in the mihi repo and are not audited here; cross-repo dep state is captured in [`development/state.md`](development/state.md), not here.
>
> **Convention adopted from cyrius / agnosticos** (2026-05-19): same tier-table shape, scaled to iam's much smaller tree (~14 markdown docs vs cyrius's ~105). One ledger row per doc; rewrite-in-place as docs change.

This is a **ledger**, not a one-time audit. Rewrite-in-place as docs change.

---

## At a glance — 2026-05-19 inventory (v0.8.0 cut)

**14 markdown docs** under `docs/` + 9 root-level files (README, CHANGELOG, CLAUDE.md, SECURITY.md, CONTRIBUTING.md, CODE_OF_CONDUCT.md, VERSION, cyrius.cyml, cyrius.lock). Bucket counts:

| Bucket | Count | What it means |
|---|---|---|
| ✅ **Fresh / touched in current cycle** | 19 | All operational + root docs were touched across the v0.5.0 → v0.8.0 arc (state.md / roadmap.md / CHANGELOG / VERSION / both audit docs / benchmarks.md / ADR 0001 (now Superseded) / ADR 0002 (now Accepted) / sample-output.txt / README / CLAUDE.md / SECURITY.md / etc.). |
| 🟡 **Stale — refresh in place** | 0 | None flagged. |
| 🟠 **Read-through outstanding** | 0 | None flagged. |
| 🔵 **Probably evergreen** | 4 | `docs/adr/template.md`, `docs/adr/README.md`, `docs/architecture/README.md`, `CODE_OF_CONDUCT.md` — convention / framing docs; re-read pass at major releases, not weekly. |
| 📦 **Archive — frozen by design** | 0 | No `docs/development/archive/` directory yet; audit docs are date-stamped artifacts (not in-place refreshed) but live in their `docs/audit/` tier, not an archive. ADR 0001 stays in `docs/adr/` with `Superseded` status — same archive-in-place pattern. |
| ❓ **Open strategic question** | 0 | **Closed** at v0.8.0 — ADR 0002 accepted (output-shape reorder to identity → runtime → hardware spine). The byte contract is now the v1.0-target shape; F-001 mitigation is the only remaining pre-v1.0 source change. |

**Why now**: iam's doc tree has been actively maintained since scaffold (every release cuts a state.md refresh and a CHANGELOG entry per CLAUDE.md), but the *aggregate* currency had no surface. This file is that surface — same convention cyrius, agnosticos, and the rest of the first-party set use.

**2026-05-19 sweep notes (v0.8.0 cut)**: ADR 0002 accepted in this cut — output-shape reorder lands. Doc-side effect: ADR 0001 status → `Superseded by 0002`; ADR 0002 status → `Accepted`; `docs/examples/sample-output.txt` regenerated; `docs/development/state.md` *Shape* and *Output* sections rewritten for the new spine; `tests/iam.tcyr` byte-exact assertions regenerated. Closes the lone ❓ Open strategic question that the v0.7.0 doc-health scaffold surfaced. No new ❓ entries opened — the next pre-v1.0 source change (F-001 mitigation) is a tracked roadmap item, not a strategic question.

**Prior sweep (2026-05-19, v0.7.0)**: initial scaffold of this file. No drift to retire — every doc was touched within the active 0.5.0 → 0.7.0 milestone-closure arc, and the M5.5 audit cut that day verified zero source drift across the same window.

---

## Tier 1 — Structural docs (root + `docs/` root)

| File | Last touched | Status | Action |
|---|---|---|---|
| `README.md` | 2026-05-19 | ✅ Fresh | Top-level project README. Touched at v0.7.0 cut. |
| `CHANGELOG.md` | 2026-05-19 | ✅ Fresh | **Source of truth per CLAUDE.md.** Through v0.7.0 (M5.5 audit closeout). Refreshed every release. |
| `CLAUDE.md` | 2026-05-19 | ✅ Fresh | Process + procedures + project-identity. Volatile state delegated to `docs/development/state.md` per its own principle. |
| `VERSION` | 2026-05-19 | ✅ Fresh | Single source of truth for version (`0.7.0` at last edit). `cyrius.cyml [package].version = "${file:VERSION}"` auto-tracks. |
| `cyrius.cyml` | 2026-05-19 | ✅ Fresh | Manifest. Toolchain pin `6.0.1`; mihi `0.7.0`; ai-hwaccel `2.2.6`. |
| `cyrius.lock` | 2026-05-19 | ✅ Fresh | Generated artifact. Refreshed by `cyrius deps` whenever pins move. |
| `SECURITY.md` | 2026-05-19 | ✅ Fresh | Threat-surface summary + reporting address. Threat model matches the M5 / M5.5 audit docs (mihi return-data abuse + TTY escape injection). |
| `CONTRIBUTING.md` | 2026-05-19 | 🔵 Evergreen | Contribution conventions; touched only when the process changes. |
| `CODE_OF_CONDUCT.md` | 2026-05-19 | 🔵 Evergreen | Standard COC; rarely changes. |
| `LICENSE` | 2026-05-19 | 🔵 Evergreen | GPL-3.0-only. Frozen by the license choice. |
| `docs/benchmarks.md` | 2026-05-19 | ✅ Fresh | Invocation-time benchmark methodology + three-point trend (v0.3.0 → M3@9df0859 → v0.5.0). M5 < 10 ms gate verified at ~1.5 ms on archaemenid. Refresh when a new benchmark run lands or when a perf-relevant change ships. **Note**: header table says "Toolchain: cyrius 6.0.0" while current pin is 6.0.1 — minor drift, refresh at next bench re-run. |

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
| `state.md` | 2026-05-19 | ✅ Fresh | **Rotates every release.** v0.7.0 — M5.5 audit closeout; Version / Audit / Next sections all refreshed at this cut. |
| `roadmap.md` | 2026-05-19 | ✅ Fresh | **Rotates per milestone.** M0 through M5.5 shipped; M6 (v0.9.0 RC + v1.0.0) is the remaining milestone, gated on F-001 mitigation + mihi 1.0. "Out of scope (for v1.0)" section restructured into honest "Not iam's job" vs "Deferred" splits at v0.6.0. |

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
| `2026-05-19-audit.md` | 2026-05-19 | 🔵 Dated artifact — M5 P(-1) audit (v0.5.0 scope). Superseded for the M5.5 scope by the doc below, kept in-place as historical record. |
| `2026-05-19-m5.5-audit.md` | 2026-05-19 | 🔵 Dated artifact — M5.5 refreshed audit (v0.6.0 scope). Verdict pass, no new findings. CVE/0day dep-tree sweep clean. F-001 still the lone v1.0 blocker; F-002 still accepted-risk INFO. |

Per CLAUDE.md *Process P(-1)*: next renewal trigger is F-001 mitigation landing in `src/display.cyr` (first source change since v0.5.0), or a mihi/ai-hwaccel major bump, or the M6 v1.0 cut — whichever comes first.

---

## Tier 6 — Guides + Examples (`docs/guides/`, `docs/examples/`)

| File | Last touched | Status | Notes |
|---|---|---|---|
| `guides/getting-started.md` | 2026-05-19 | ✅ Fresh | Build / layout / "adding a display line" walk-through. References the scaffold-era source layout — verify the "Once M1+ ships" section is no longer hedging (M1 / M2 / M3 / M4 all shipped) at next touch. |
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
| 2 | **Periodic audit** — full P(-1) source review + dep-tree CVE sweep before the M6 v1.0 freeze (and on any mihi / ai-hwaccel / cyrius major bump). | Before v1.0; on dep-major bump | [`CLAUDE.md`](../CLAUDE.md) *Process P(-1)* | Most recent: 2026-05-19 (M5 → M5.5). Next trigger: F-001 mitigation landing, since that's the first source change since v0.5.0 — see the M5.5 audit's *Next audit trigger* clause. |
| 3 | **ADR 0002 resolution** — **CLOSED at v0.8.0 (Accepted)**. Output-shape reorder landed; ADR 0001 §2-§3 superseded; tests regenerated; sample-output regenerated. Row kept for one cycle as historical commitment, removable at next sweep if uneventful. | (resolved) | [`adr/0002-output-shape-reorder.md`](adr/0002-output-shape-reorder.md) | Closed. |
| 4 | **Benchmark refresh** — re-run the bench harness against current `build/iam` whenever a perf-relevant change ships (F-001 sanitizer pass is a candidate — adds a per-byte filter loop to renderers). | On perf-relevant change | [`benchmarks.md`](benchmarks.md) | Current trend: 3 points (v0.3.0 / M3@9df0859 / v0.5.0). Header table also pins "cyrius 6.0.0" while VERSION-aligned toolchain is 6.0.1 — refresh at next bench run. |

---

*Initial scaffold: 2026-05-19 (v0.7.0 — M5.5 audit closeout). Convention pattern matches `cyrius/docs/doc-health.md` (and the broader first-party set: agnosticos / vidya / phylax / etc.), scaled to iam's ~14-doc tree. Refresh in place when docs are touched.*
