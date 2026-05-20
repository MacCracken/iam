# iam — invocation-time benchmarks

> Login-path tool. Every shell session pays this cost; the M5 gate is
> **cold-start < 10 ms** on the reference host. Methodology, four-point
> trend (v0.3.0 / M3 / v0.5.0 / v0.9.0), and the M5 verdict live here.

## Reference host

`archaemenid` — the maintainer's daily-driver laptop.

| Field          | Value                                       |
| -------------- | ------------------------------------------- |
| CPU            | AMD Ryzen 7 5800H with Radeon Graphics (8c/16t) |
| Memory         | 59 GiB                                      |
| Kernel         | Linux 7.0.5-arch1-1                         |
| Distro         | Arch Linux                                  |
| GPU            | AMD Radeon integrated (PCI 0x1002:0x1638)   |
| Toolchain      | cyrius 6.0.1                                |

All numbers below are from this host. A different machine will produce
different absolute timings; the contract is the **shape** of the cost
(probe-dominated, single-digit ms) and the **gate** (< 10 ms cold).

## Methodology

`iam` is invoked per-login, so the benchmark we care about is
**end-to-end process invocation time**, not in-process micro-bench.
That rules out the `lib/bench.cyr` framework (which measures
in-process work and excludes process startup).

The measurement is a shell loop over N separate `execve` invocations,
total wall time captured via `date +%s%N`, divided by N:

```sh
N=500
START=$(date +%s%N)
for i in $(seq 1 $N); do ./build/iam > /dev/null; done
END=$(date +%s%N)
echo "$(( (END - START) / N / 1000 )) us/invocation"
```

Three trials per build to surface jitter. Output redirected to
`/dev/null` so terminal-render cost isn't counted (we want the
program's cost, not the terminal's). Stdout `(unknown)` fallbacks
suppressed via TTY-equivalence — the work done is identical with or
without a TTY (ADR 0001 §5, carried over by ADR 0002).

**Why not `lib/bench.cyr`**: the bench library is the right tool for
in-process hot loops where `clock_gettime` overhead matters. Here the
unit of work *is* the process — fork-exec, dynamic relocation (none —
we're statically linked), page-in, run, exit. Measuring inside the
process would miss the very costs that gate the M5 < 10 ms target.

**Why batch then divide**: a single-shot `date +%s%N` around one
invocation captures ~50–500 us of shell jitter on top of the
sub-ms program work; that's the wrong order of magnitude for a
tool whose cold-start budget is single-digit milliseconds. Batching
500 invocations amortizes the jitter into noise.

## Four-point trend

Four milestone builds along the v0.1 → v0.9 line, same host, same
methodology, same N=500. Numbers are the median of three trials.

| Build                 | Per-invocation | Lines emitted | Notes                                          |
| --------------------- | --------------:| -------------:| ---------------------------------------------- |
| v0.3.0 (M1+M2)        |     **555 us** |             6 | CPU/Memory/Kernel/Host/Distro/Uptime, per-line write(2) |
| v0.4.0-eq (M3, 9df0859) |   **1514 us** |             7 | + GPU line, per-line write(2)                  |
| v0.5.0 (M4)           |    **1511 us** |             7 | + render-into-buffer, single write(2)          |
| v0.9.0 (M6 RC)        |    **1510 us** |             7 | + ADR 0002 reorder, + F-001 sanitizer (`iam_copy_value` per-byte filter on value-column bytes) |

**Verdict**: well under the 10 ms M5 gate at every measured point. The
budget is comfortable — `iam` adds about a millisecond and a half to
shell login.

### What the trend says

1. **GPU detection dominates** (M2 → M3 jump). Going from 6 lines to
   7 lines tripled per-invocation cost from ~555 us to ~1.5 ms. The
   probe (`mihi_gpu_count` + `mihi_gpu_name`) reads PCI config space
   and matches against an in-binary device table — the actual *line*
   in the output is cheap; the *probe* behind it is the M3 budget
   we paid for the GPU label.
2. **The M4 single-flush refactor didn't move the needle.** Going
   from ~14 `write(2)` syscalls to 1 saved syscall overhead but at
   this scale the writes weren't the bottleneck — the probe work
   was. The refactor is still right (testability + ADR-contract
   observability, see CHANGELOG v0.5.0), but the perf win was
   nominal, not load-bearing. Recording this honestly so future
   maintainers don't assume the refactor was speed-motivated.
3. **The trend is flat between M3, M4, and v0.9.0** — all three runs
   land in the 1490–1545 us range. That's the noise floor for this
   methodology on this host. If a future change shifts the median
   outside that band, that's a real signal worth investigating.
4. **F-001 sanitizer is free at this scale.** Adding `iam_copy_value`
   (per-byte `< 0x20 → '?'` filter on value-column bytes through
   `iam_render`, `iam_render_buf`, `iam_render_kernel`) costs one
   comparison + one possible store-overwrite per value byte.
   Real-world value lengths are ~10-50 bytes per line; the total
   added work is ~250 bytes of comparisons per invocation against
   a ~1500 µs probe budget. The median didn't move — landed
   identical to v0.5.0's 1511 µs within the noise band. F-001
   mitigation pays nothing measurable on the login hot-path.

### Implications for future cuts

- Adding more probes is the dominant cost lever. The output-shape
  contract is frozen at v1.0 (ADR 0002, which supersedes ADR 0001
  §2-§3), so this is bounded — six required lines + one optional.
  No "v1.0 perf surprise" landing here.
- If the < 10 ms gate is ever threatened, the place to look is the
  individual mihi probes, not iam's renderer or syscall pattern.
  Profile mihi, not iam.

## Reproducing

From the repo root with a built binary:

```sh
cyrius build src/main.cyr build/iam
N=500
for trial in 1 2 3; do
  START=$(date +%s%N)
  for i in $(seq 1 $N); do ./build/iam > /dev/null; done
  END=$(date +%s%N)
  printf "trial=%d per=%dus\n" $trial $(( (END - START) / N / 1000 ))
done
```

Expect numbers within ~10 % of the v0.9.0 row above on comparable
hardware. Bigger deviations are worth filing.

## History

- **2026-05-19** — initial three-point capture (v0.3.0, M3@9df0859,
  v0.5.0). Methodology established. M5 < 10 ms gate verified with
  ~6.5× headroom.
- **2026-05-19** — fourth point added at v0.9.0 (M6 RC).
  Re-measurement after the ADR 0002 reorder (v0.8.0) and the F-001
  sanitizer (`iam_copy_value` at the renderer boundary). Median
  1510 us — sits inside the v0.5.0 noise floor (1490–1545 us).
  Sanitizer is invisible at this scale. Toolchain row bumped
  cyrius 6.0.0 → 6.0.1 (the active local pin).
