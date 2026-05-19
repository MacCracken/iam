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
  CPU / RAM / kernel / uptime / distro / hostname probes (currently
  scaffolded; not yet a published dep).
- Renders via [`darshana`](https://github.com/MacCracken/darshana)
  ANSI primitives if any color is ever desired.
- One redraw per invocation. Login shell calls `iam`; output flushes;
  process exits. No daemon, no cache.

## Status

Pre-1.0 scaffold (0.1.0). Prints version and exits.

## Build

```sh
cyrius deps                           # resolve stdlib
cyrius build src/main.cyr build/iam   # compile
./build/iam                            # prints "iam v0.1.0 — scaffold"
cyrius test                            # run tests/*.tcyr
```

## License

GPL-3.0-only
