# Contributing to iam

Contributions are welcome. All contributions must be licensed under
GPL-3.0-only.

## Development

Follow the conventions in [`CLAUDE.md`](CLAUDE.md) and the AGNOS
[first-party standards](https://github.com/MacCracken/agnosticos/blob/main/docs/development/planning/first-party-standards.md).

Build and test before submitting:

```sh
cyrius deps
cyrius build src/main.cyr build/iam
cyrius test
```

## Before proposing a feature

`iam` is intentionally minimal. Before opening a PR for a new
feature, please check:

- Is this a new **probe** (a new system fact)? **It goes in
  [mihi](https://github.com/MacCracken/mihi), not iam.** iam is
  the presentation layer over mihi probes.
- Is this a **configurability** feature (themes, plugins, env-var
  toggles, output formats)? **It's out of scope** — see
  `docs/development/roadmap.md` § "Out of scope" for why.
- Is this an **ASCII logo / banner** feature? Use
  [BannerManor](https://github.com/MacCracken/bannermanor).

If the feature genuinely fits `iam`'s scope (a display refinement,
a bug fix, a pipe-detection improvement), proceed.

## Adding a display line

1. Confirm mihi has the probe you need.
2. Add the line emission to `src/main.cyr` / `src/display.cyr`.
3. Happy + degraded-output tests.
4. CHANGELOG entry. Mark `Breaking` if you alter line order.

## Reporting Issues

Open an issue at https://github.com/MacCracken/iam/issues.

For security-sensitive issues, see [`SECURITY.md`](SECURITY.md).
