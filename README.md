# iam

`fastfetch` / `neofetch`-equivalent system-info display for login
MOTD and screenshot flex — in
[Cyrius](https://github.com/MacCracken/cyrius).

Pure inverse of `whoami`: `whoami` says who *the user* is; `iam` says
what *the system* is. Tiny, evocative, reads as a sentence when invoked.

## Design principle: keep it whoami-simple

`whoami` prints the user. `iam` prints the system. Same minimalism.

- **NO theming engine.**
- **NO plugin system.**
- **NO 50-line configurable output.**
- **NO ASCII-logo-of-the-month-club.**

Just the essential facts the box has to declare about itself. Resist
all "make it more like neofetch" feature creep. The shared probe lib
([mihi](https://github.com/MacCracken/mihi)) does the work; `iam` is
a thin presentation surface over it.

## Positioning

Fifth member of the terminal-aesthetics set:

- [`commandress`](https://github.com/MacCracken/commandress) (`cmdrs`) — prompt rendering
- [`darshini`](https://github.com/MacCracken/darshini) — file listing display
- [`hapi`](https://github.com/MacCracken/hapi) — dotfile / symlink management
- [`BannerManor`](https://github.com/MacCracken/bannermanor) (`bnrmr`) — ASCII banner generation
- **`iam`** — system info / login MOTD / screenshot flex

## Shape

- Consumes [`mihi`](https://github.com/MacCracken/mihi) for the
  CPU / RAM / GPU / kernel / uptime / distro / hostname probes —
  a published dep, pinned in `cyrius.cyml`.
- No color today, and no TTY detection: output is byte-identical to a
  terminal and to a pipe, so `iam | awk ...` sees exactly what you see.
  Deferred rather than ruled out — [ADR 0001 §5](docs/adr/0001-output-shape.md)
  keeps color open for v2.0 behind its own ADR, defaulting to off.
- One redraw per invocation. Login shell calls `iam`; output flushes;
  process exits. No daemon, no cache.

## Status

**v1.1.6.** The output shape froze at v1.0.0 — line order, label set,
label width, the `(unknown)` fallback, and exit-0 discipline are a
contract now, and changing any of them is a major-version event. See
[ADR 0002](docs/adr/0002-output-shape-reorder.md).

Six required lines, plus a seventh `GPU:` line when an accelerator is
detected:

```
Distro: Arch Linux
Host:   archaemenid
Kernel: Linux 7.1.8-arch1-3
Uptime: 1d 1h 6m
CPU:    AMD Ryzen 7 5800H with Radeon Graphics
GPU:    AMD Radeon (PCI 0x1002:0x1638) [3 GiB]
Memory: 59 GiB
```

A probe that fails renders `(unknown)` rather than an error, and the
exit code stays 0 — a login MOTD running under `set -e` must not trip
the shell.

## Build

```sh
cyrius deps                           # resolve stdlib + mihi + ai-hwaccel
cyrius build src/main.cyr build/iam   # compile
./build/iam                           # print the system card
cyrius test tests/iam.tcyr            # run the suite
```

## License

GPL-3.0-only
