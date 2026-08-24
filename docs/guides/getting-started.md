# Getting started with iam

## Build

```sh
cyrius deps                          # resolve stdlib + mihi + ai-hwaccel
cyrius build src/main.cyr build/iam  # compile
./build/iam                          # print the system card
cyrius test tests/iam.tcyr           # run the suite
```

`cyrius lib sync --full` re-vendors `lib/` from the pinned toolchain
snapshot. You need it after bumping `[package].cyrius` — the build
warns (`./lib/ shadows version-pinned ...`) when `lib/` has drifted
from the pin.

## Layout

- `src/main.cyr` — driver: opens the shared uts buffer, calls the
  mihi probes, accumulates rendered lines, flushes once
- `src/display.cyr` — line formatter (label-padded output), byte
  formatting, the value-column sanitizer
- `src/uptime.cyr` — seconds → human format
- `tests/iam.{tcyr,bcyr,fcyr}` — tests / benchmarks / fuzz
- `lib/` — vendored; managed by `cyrius deps` / `cyrius lib sync`,
  never hand-edited

## Adding a display line

Read the feature-creep gate in `CLAUDE.md` first — the answer is
usually "this belongs in a sibling tool." If it genuinely belongs
here:

1. Confirm mihi has a probe for the fact you want. **If it doesn't,
   add the probe to mihi first** — `iam` does not probe directly.
2. Add the line to `src/main.cyr` / `src/display.cyr`.
3. Add a happy-path test + a probe-returns-error test in
   `tests/iam.tcyr`. The error path should produce `unknown` in
   the output line, not crash.
4. Update `docs/examples/sample-output.txt` if it exists.
5. CHANGELOG entry under `Added`.

**Since v1.0.0 this is a bigger deal than the list suggests.** ADR
0002 froze the line order, the label set, the label width, the
`(unknown)` fallback, and exit-0 discipline. Adding, removing, or
reordering a line is a **major-version** change, not an `Added`
entry. An optional line that slots into the existing spine without
disturbing it (the way `GPU:` does) needs its own ADR — see
[ADR 0003](../adr/0003-gpu-line-memory-suffix.md) for the shape of
that argument — and the CI smoke gate's six-or-seven-line count
check has to be updated in lockstep.

## Why so few features?

`iam` is intentionally minimal — see `CLAUDE.md` § "Key Principles"
and `docs/development/roadmap.md` § "Out of scope." Most "improvements"
to neofetch/fastfetch belong in a sibling tool, not in `iam`.

See [`../adr/template.md`](../adr/template.md) when a non-trivial
design choice (especially anything that touches output shape)
deserves an ADR.
