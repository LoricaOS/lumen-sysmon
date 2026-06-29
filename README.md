# lumen-sysmon

The system monitor for **AspisOS**, a capability-based, no-ambient-authority
operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

sysmon is a standalone client of the [lumen](https://github.com/AspisOS/lumen)
compositor, speaking the Lumen external window protocol (the same pattern as
calculator, settings and terminal). It is a component of the Lumen desktop,
distributed as a [herald](https://github.com/AspisOS/AspisOS) package and
installed into the `/apps` bundle tree.

## Role in the system

- A task / resource monitor that reads `/proc` directly — it requires no new
  kernel support:
  - `/proc/meminfo` — `MemTotal` / `MemFree`, drawn as a memory bar (green /
    yellow / red at 70% / 90%).
  - `CLOCK_MONOTONIC` — uptime, shown as `up Xh Ym Zs`.
  - `readdir("/proc")` — the all-digit entries are pids.
  - `/proc/<pid>/status` — `Name` / `State` / `VmSize`, shown in the table.
- The size column reports `VmSize` (sum of VMA lengths, in kB), labelled
  `MEM(KB)`. Aegis procfs does not yet expose `VmRSS` (no resident-set
  accounting), so sysmon mirrors what `ps` reads. The list is sorted by `VmSize`
  descending, pid ascending on ties.
- Refreshes at about 1 Hz: a 1000 ms `lumen_wait_event` timeout re-scans
  `/proc`; any input also forces a re-render.
- Input: arrow keys (raw or Lumen's synthetic `0xF1`/`0xF2`) move the selection,
  a left click selects a row, the wheel and a scrollbar scroll the list, and `k`
  sends `SIGTERM` to the selected pid. pid 0 and pid 1 (init) are guarded against
  signalling.
- The window is 720x560, clamped to the framebuffer reported via `LUMEN_FB_W` /
  `LUMEN_FB_H`. The process table is capped at 256 entries (truncation is logged).

## Capabilities

sysmon's cap policy (`pkg/etc/aegis/caps.d/sysmon`) is the baseline:

```
service
```

It needs no elevated capability beyond the default service profile: it reads
`/proc` and connects to the compositor over the Lumen socket. Sending `SIGTERM`
to a selected process uses the ordinary `kill` path within that profile; it holds
no AUTH, SETUID, FB or network capability.

Although its herald package id (`lumen-sysmon`) differs from the bundle/exec name
(`sysmon`) and it installs an `/apps` binary plus a cap policy, that naming and
cross-tree install make it a `class=system` package: first-party and
signature-trusted, installed verbatim by herald.

## Building

sysmon fetches a pinned [glyph](https://github.com/AspisOS/glyph) toolkit
artifact (the GUI libraries it links: glyph + libaudio + libauth) and builds
against it, then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the toolkit release fetched by `tools/fetch-glyph.sh`.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption — point it
  at an Aegis-native `cc` to build on-device in the future).
- `HERALD_KEY` signs the `.hpkg`.

Output: `lumen-sysmon.hpkg` (a `class=system` herald package) +
`lumen-sysmon.hpkg.sig`.

## Package payload

```
/apps/sysmon/sysmon                  the system monitor binary
/apps/sysmon/app.ini                 the Lumen app-bundle manifest (name=System Monitor, exec=sysmon)
/etc/aegis/caps.d/sysmon             its capability policy (service)
```

## Repository layout

```
src/        sysmon source
pkg/        install-tree skeleton shipped verbatim (apps/ bundle + caps.d)
tools/      fetch-glyph.sh (toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this component's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — sysmon is a client of the
[lumen](https://github.com/AspisOS/lumen) compositor, so installing it pulls
lumen (which in turn provides the desktop fonts).

## Sibling components

- [bastion](https://github.com/AspisOS/bastion) — display manager / login greeter
- [lumen-imageviewer](https://github.com/AspisOS/lumen-imageviewer) — image viewer
- [lumen-netman](https://github.com/AspisOS/lumen-netman) — network manager
