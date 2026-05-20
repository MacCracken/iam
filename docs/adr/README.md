# Architecture Decision Records

Decisions about iam — what we chose, the context, and the consequences we accept. Use these when a future reader would reasonably ask *"why did we do it this way?"*

## Conventions

- **Filename**: `NNNN-kebab-case-title.md`, zero-padded to four digits. Never renumber.
- **One decision per ADR.** If a decision supersedes a prior one, add a new ADR and set the old one's status to `Superseded by NNNN`.
- **Status lifecycle**: `Proposed` → `Accepted` → (optionally) `Superseded` or `Deprecated`.
- Use [`template.md`](template.md) as the starting point.

## ADR vs. architecture note vs. guide

| Kind | Lives in | Answers |
|---|---|---|
| ADR | `docs/adr/` | *Why did we choose X over Y?* |
| Architecture note | `docs/architecture/` | *What non-obvious constraint is true about the code?* |
| Guide | `docs/guides/` | *How do I do X?* |

## Index

- [0001 — Output shape](0001-output-shape.md) — line order, label
  column, fallback policy, optional-GPU rule, no-color / pipe
  equivalence, exit-0 contract. (Superseded by 0002, 2026-05-19.)
- [0002 — Output shape reorder](0002-output-shape-reorder.md) —
  identity → runtime → hardware spine (Distro / Host / Kernel /
  Uptime / CPU / GPU? / Memory). Supersedes ADR 0001 §2-§3 only;
  §1, §4, §5, §6 carry over unchanged. (Accepted, 2026-05-19.)
