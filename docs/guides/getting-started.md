# Getting started with iam

## Build

```sh
cyrius deps                          # resolve stdlib (+ mihi from M1 onward)
cyrius build src/main.cyr build/iam  # compile
./build/iam                           # prints scaffold version line
cyrius test                           # run tests/*.tcyr
```

## Layout

- `src/main.cyr` — entry point; CLI dispatch + line emission
- `tests/iam.{tcyr,bcyr,fcyr}` — tests / benchmarks / fuzz

Once M1+ ships:

- `src/display.cyr` — line formatter (label-padded output)
- `src/uptime.cyr` — seconds → human format
- `[deps.mihi]` wired in `cyrius.cyml`

## Adding a display line

1. Confirm mihi has a probe for the fact you want. **If it doesn't,
   add the probe to mihi first** — `iam` does not probe directly.
2. Add the line to `src/main.cyr` / `src/display.cyr`.
3. Add a happy-path test + a probe-returns-error test in
   `tests/iam.tcyr`. The error path should produce `unknown` in
   the output line, not crash.
4. Update `docs/examples/sample-output.txt` if it exists.
5. CHANGELOG entry under `Added`. Mark `Breaking` if it changes the
   established line order.

## Why so few features?

`iam` is intentionally minimal — see `CLAUDE.md` § "Key Principles"
and `docs/development/roadmap.md` § "Out of scope." Most "improvements"
to neofetch/fastfetch belong in a sibling tool, not in `iam`.

See [`../adr/template.md`](../adr/template.md) when a non-trivial
design choice (especially anything that touches output shape)
deserves an ADR.
