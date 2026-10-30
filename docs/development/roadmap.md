# iam — Roadmap

> Open work only, grouped into releases. Shipped work lives in
> [`../../CHANGELOG.md`](../../CHANGELOG.md), live state in
> [`state.md`](state.md). An item leaves this file when it ships; one
> that slips moves to a later release rather than being marked done
> here.

Every 1.x release keeps the v1.0 output contract: line order, label
width, label spelling, the `(unknown)` fallback and exit 0, per
[ADR 0002](../adr/0002-output-shape-reorder.md) and
[ADR 0003](../adr/0003-gpu-line-memory-suffix.md). A change to any of
them is a major-version bump.

## Patch work — no minor of its own

Testing, verification, audit and docs ship in patch releases on
whatever line is current.

### Verification on real targets

Several things `state.md` says about iam are true by compiling, not by
running, on the current pins.

- **AGNOS.** Run the `--agnos` build under QEMU through agnsh on the
  current pins and record which lines render. iam last rendered on
  AGNOS at 1.1.5 (kernel 1.45.10); 1.1.6 through 1.2.1 were
  compile-checked only. mihi 1.3.2's AGNOS fixes (B-8 CPUID max leaf,
  B-9 sysinfo bounds, B-10 uname seal, B-13 `mihi_distro` into `buf`)
  land on iam's CPU, Memory, Uptime and Distro lines there.
- **arm64 hardware.** Run `iam` and `tests/iam.tcyr` on an arm64 Linux
  host. Every cut so far has only compiled `--aarch64`.
- **Accelerator hosts.** The `GPU:` line has only been seen on
  archaemenid (AMD iGPU through ROCm). Run iam on:
  - an NVIDIA or Intel GPU host (the Vulkan placeholder sizes, 1.1.9);
  - a Core Ultra or Ryzen AI host (the `/sys/class/accel` driver
    routing that replaced the phantom TPU, 1.2.1);
  - a hybrid-graphics laptop on kernel 7.3+, reading the dGPU's
    `power/runtime_status` before and after an `iam` run (mihi audit
    U-1).
- **Benchmark re-baseline.** The 1.2.0 and 1.2.1 A/Bs ran at load
  average 7–15, so their absolute figures are not comparable to the
  earlier rows in `docs/benchmarks.md`. Re-run on a quiet box and
  record the 1.2.1 cost (+18–24 % under load) as an absolute number.
- **Acceptance**: each run recorded in `state.md` with date, host and
  pins.

### Tests and tooling

- **Fuzz harness.** `tests/iam.fcyr` is a stub. Fuzz the display path:
  `iam_copy_value` (sanitizer, `IAM_VALUE_MAX` clamp, UTF-8 back-off)
  and the four renderers, with canary bands around the output buffer,
  the way mihi 1.3.2's `tests/mihi.fcyr` does for its parsers.
- **Benchmark harness.** `tests/iam.bcyr` times a no-op. Bench the
  renderers in-process, and make the wall + CPU invocation harness used
  at 1.2.0 and 1.2.1 the one `docs/benchmarks.md` *Reproducing* gives.
- **CI parity with mihi 1.3.2**: a fuzz step, `cyrius lint --strict`,
  and `tests/*.fcyr` in the fmt sweep.

### Audit

- **P(-1) audit on the current pins.** The last full audit is v1.1.7
  (cyrius 6.5.35, mihi 1.2.4). Since then: toolchain moves to 6.7.18,
  mihi 1.2.5–1.3.2 (including its own 14-fix sweep), and ai-hwaccel
  2.3.18 → 2.5.5. The re-audit triggers in `state.md` name major bumps
  only, so none fired. File it in `docs/audit/`.

### Docs

- **doc-health re-sweep.** Rows not touched since the v0.9.0-era sweep
  have not been re-read; the ledger says so itself. Re-read each one
  and re-date it.
- **README.md, `docs/guides/getting-started.md` and
  `docs/examples/sample-output.txt`** checked against the current pins
  and output. Part of the sweep, called out because they are the first
  files a user reads.

## 1.3.x — GPU line accuracy

What the `GPU:` line prints and what producing it costs. Each item is
gated upstream and lands with the mihi pin that carries the fix.

- **Placeholder GPU sizes print as measured.** ai-hwaccel's figures for
  NVIDIA / Intel (Vulkan scan: 4 / 2 GiB), Cerebras (44 GiB), Graphcore
  (900 MiB) and Gaudi / Neuron (by generation) are not read from the
  device, and iam prints them in the ADR 0003 `[N GiB]` suffix. ADR
  0003 already prints the bare device name when the size is 0.
  *Gate: mihi / ai-hwaccel reporting an unknown size as 0, or saying
  whether a size was measured.* iam-side handling instead would need
  an ADR 0003 amendment.
- **dGPU wake on every invocation** (mihi audit U-1). *Gate:
  ai-hwaccel's no-wake PCIe path (mihi roadmap 1.6.x).*
- **Probe cost of ai-hwaccel 2.5.5's no-exec path** (+18–24 % per
  invocation at 1.2.1). *Gate: upstream ai-hwaccel / mihi.*

## 1.x.x — Gated, no release assigned

- **F-002**: `strlen` on mihi cstrings is trust-dependent;
  `IAM_VALUE_MAX` bounds the copy, not the scan. *Gate: an `n`-limited
  probe variant in mihi.*

## Out of scope

The list keeps future contributors from adding to iam by accident.
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
  *Deferred* list below under Color / shell-theme consumption.
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
  semantics) — not iam shipping its own themes. The v1.0 byte
  contract froze on the no-color baseline, so color lands as
  opt-in and plain bytes stay the default pipe target.
- **Network probes** (e.g. local IP, link state) — deferred until
  kernel-side bring-up is operational. Adds a real probe surface
  to mihi first; iam adopts after.
- **Runtime state** (Battery %, Swap used, Disk used%) — a v2.0
  conversation. v1.0 froze the identity → kernel → uptime →
  hardware set; runtime-state lines would be additive on top of
  it.

## Cross-references

- [`state.md`](state.md) — live status (mihi version pin, output lines)
- [`../../CHANGELOG.md`](../../CHANGELOG.md) — release history
- [mihi roadmap](https://github.com/MacCracken/mihi/blob/main/docs/development/roadmap.md) — where most gated items here resolve
