# WebKit Exploit — Console

A single-shot PlayStation 4 WebKit exploit harness with a built-in on-screen console, real-time PASS/FAIL status reporting, and `.txt` log export. The exploit builds a fake host object to obtain an arbitrary read/write primitive, then optionally performs a kernel jailbreak.

**Author tag:** `OptiTronOffical`
**Revision:** `genuine-u8-rw-3-1352`

> ⚠️ **For research and educational use only.** Only run this on hardware you personally own. Kernel-level memory corruption can brick your console or cause data loss.

---

## Table of Contents

1. [What It Does](#what-it-does)
2. [Features](#features)
3. [Requirements](#requirements)
4. [Quick Start](#quick-start)
5. [URL Parameters](#url-parameters)
6. [Reading the Output](#reading-the-output)
7. [Log Export](#log-export)
8. [How It Works](#how-it-works)
9. [Supported Firmware](#supported-firmware)
10. [Troubleshooting](#troubleshooting)

---

## What It Does

The page runs a WebKit memory-corruption exploit that produces a genuine arbitrary read/write primitive. When the primitive is verified, the page **halts by default** and reports the result. The kernel jailbreak stage is opt-in via a URL parameter so that a failed exploit cannot trap the console in a crash/reload loop.

The two stages are:

| Stage | What it does | When it runs |
|-------|--------------|--------------|
| **R/W primitive** | Builds a fake JSObject host, leaks object addresses, and demonstrates read + write through a corrupted `Uint8Array` | Always |
| **Kernel jailbreak** | Hijacks `sysent`, patches `ucred`, and swaps the file descriptor root vnode | Only when `?jb=1` is present |

---

## Features

- **Live console** — every log line is timestamped and color-coded by severity (pass / fail / warn / info).
- **Status bar** — shows the current state at a glance: `INITIALIZING`, `RUNNING`, `PASS`, `FAIL`, or `HALTED`, with live PASS/FAIL/WARN counters.
- **Single-shot execution** — no auto-retry loop. One attempt, one clean terminal state.
- **Jailbreak gate** — the kernel stage is disabled unless you explicitly opt in.
- **Log export** — download the full session as a timestamped `.txt`, copy to clipboard, or auto-save on completion.
- **Firmware auto-detection** — reads the PS4 firmware version from the user agent and selects the correct kernel offsets.

---

## Requirements

- A PlayStation 4 whose firmware is listed in [Supported Firmware](#supported-firmware).
- A way to load the HTML file on the console (e.g. a local web server, USB-hosted page, or a host payload loader).
- A browser/user-agent string that includes the firmware version, e.g. `PlayStation 4/9.00`.

---

## Quick Start

### 1. Basic run (R/W primitive only — safe)

Load the page with no parameters:

```
jb.html
```

The exploit runs, and on success you will see:

```
N-rw-GENUINE-UB-RW-PASS-carrier-vector-restored=true
N-rw-HALTED-BEFORE-JAILBREAK-rw-primitive-ready-no-kernel-writes-performed
N-rw-HALT-NOTE-append-?jb=1-to-url-to-run-kernel-jailbreak
```

The status bar turns **amber** with `R/W PASS — HALTED`. No kernel memory is touched.

### 2. Full run (R/W primitive + kernel jailbreak)

Append `?jb=1`:

```
jb.html?jb=1
```

On success you will see:

```
N-rw-JAILBREAK-GATE-explicitly-enabled-by-url-param
N-rw-===== PS4 JAILBREAK START =====-fw=9.00
...
N-rw-===== JAILBREAK COMPLETE =====-PS4 9.00 root=true-fs=true
```

The status bar turns **green** with `JAILBREAK COMPLETE — PASS`.

### 3. Debug run

Add `?dbg=1` alongside `?jb=1` to enable verbose kernel scan logging:

```
jb.html?jb=1&dbg=1
```

This adds `SCAN-HIT`, `UCRED-SIG-CANDIDATE`, and (on failure) a `PROC-DUMP` block.

---

## URL Parameters

| Parameter | Values | Default | Description |
|-----------|--------|---------|-------------|
| `jb` | `1` | *(unset)* | Enable the kernel jailbreak stage. Without this, the exploit halts after the R/W primitive. |
| `n` | `64`–`1024` | `128` (fallback `512`) | Number of 64 KiB `ArrayBuffer`s to allocate during heap grooming. Clamped to the valid range. |
| `l` | `1` | *(unset)* | Reserved "lite" mode flag. |
| `dbg` | `1` | *(unset)* | Verbose kernel scan logging inside `runJailbreak()`. |

**Examples**

```
jb.html                                  # R/W only, halt after pass
jb.html?jb=1                             # full run
jb.html?jb=1&n=512                       # full run, larger groom
jb.html?jb=1&dbg=1                       # full run with debug dump
jb.html?n=1024                           # R/W only, max drain
```

---

## Reading the Output

### Status bar

| Color | State | Meaning |
|-------|-------|---------|
| Grey | `INITIALIZING` | Page loaded, exploit not yet running |
| Blue (pulsing) | `RUNNING` | Exploit in progress |
| Green | `PASS` | R/W primitive verified, or jailbreak complete |
| Amber | `HALTED` | R/W primitive passed; jailbreak skipped (no `?jb=1`) |
| Red | `FAIL` | Terminal failure — see the console for the reason |

### Console line levels

| Level | Meaning | Examples |
|-------|---------|----------|
| **pass** | A stage succeeded | `GENUINE-UB-RW-PASS`, `JAILBREAK COMPLETE`, `SANDBOX-ESCAPE success` |
| **fail** | A stage failed terminally | `FAILED-TERMINAL`, `HEADER-MISMATCH`, `LOAD-THREW`, `JAILBREAK-ERROR` |
| **warn** | Non-fatal anomaly or halt | `HALTED-BEFORE-JAILBREAK`, `SSV-PLACEMENT-MISS`, `NO-RESULT` |
| **info** | Normal progress | `BOOT`, `SSV-BUILT`, `ADDROF-POINTERS`, `ROOTS-LIVE` |

### Key milestone tags

| Tag | Meaning |
|-----|---------|
| `BOOT` | Exploit started; shows revision, `k`, `n`, and jailbreak gate state |
| `ADDROF-POINTERS` | Leaked host and target object addresses (`HOST=`, `TARGET=`) |
| `FAKE-ADDRESS` | Computed the fake host address (`fake = host + 0x10`) |
| `ARBITRARY-READ-PASS` | Read primitive verified |
| `ARBITRARY-WRITE-PASS` | Write primitive verified and restored |
| `GENUINE-UB-RW-PASS` | Both primitives confirmed — R/W is live |
| `HALTED-BEFORE-JAILBREAK` | R/W passed; kernel stage skipped |
| `JAILBREAK COMPLETE` | Kernel jailbreak finished successfully |

---

## Log Export

The toolbar at the top of the page provides three actions:

| Button | Action |
|--------|--------|
| **⬇ Download log (.txt)** | Saves the full session as `ps4-exploit-log-<timestamp>.txt` |
| **Copy log** | Copies the same text to the clipboard |
| **Clear** | Empties the console (does not affect the exploit) |

There are also two checkboxes:

- **auto-scroll** — keep the newest line visible (on by default).
- **auto-save .txt on finish** — automatically download the log when a terminal state (PASS, HALTED, or FAIL) is reached.

### Log file contents

The exported `.txt` contains a header block followed by every console entry:

```
==================================================
 WebKit Exploit Log — OptiTronOffical
==================================================
Revision     : genuine-u8-rw-3-1352
Firmware     : 9.00
User Agent   : PlayStation 4/9.00 ...
Started      : 2026-...
Ended        : 2026-...
Jailbreak    : DISABLED (halt before runJailbreak)
Final Status : R/W PASS — HALTED  [HALTED]
Counts       : PASS=8  FAIL=0  WARN=2  INFO=47
--------------------------------------------------
[HH:MM:SS.mmm] [INFO] 1-rw-BOOT-...
...
--------------------------------------------------
END OF LOG (57 entries)
```

---

## How It Works

### Stage 1 — Heap grooming and fake host

1. **Drain** — Allocate `DRAIN_COUNT` 64 KiB `ArrayBuffer`s to stabilize the heap.
2. **Slab transfer** — Send a 4 MiB buffer through a `MessageChannel` to free it on the other side.
3. **Hole carving** — Free butterfly holes, separators, guards, and a predecessor buffer at controlled offsets.
4. **Fake host** — Build a `fakeHost` object whose `q0` slot contains an encoded `ArrayBuffer` header and whose `q2` / `q3` slots point at a `Uint8Array` and a length word.
5. **Serialize** — Store a graph into `history.replaceState` with a duplicate reference at index `2`.

### Stage 2 — Address leak (addrof)

The exploit leaks object addresses via a `Symbol.prototype.toString` trick:

- A `Leaker` class returns `super.foo`, which triggers a proxy `get` on the prototype.
- The proxy returns the receiver, leaking the scope object.
- A getter is installed on the leaked scope, and `Object(scope.g)` wraps it.
- Calling `symbolToString` on that wrapper copies the getter's arguments (including the host and target pointers) into a string.
- The string's UTF-16 characters are read back to reconstruct the 64-bit host and target addresses.

### Stage 3 — R/W primitive

- The predecessor buffer is filled with copies of `fakeAddress` so the allocator reuses it as a fake `ArrayBuffer` cell.
- Loading `history.state` pulls the corrupted cell back into JS.
- The exploit verifies the fake cell header, upgrades `fakeHost.q0`, and points the carrier's vector at the target cell.
- A round-trip read (`rwView[0] === 0xa5`) and write (`rwView[0] = 0x5a`) through both views confirms the primitive.
- The original vector is restored so the heap stays consistent.

### Stage 4 — Kernel jailbreak (opt-in)

Only runs with `?jb=1`:

1. **Hijack `sysent[661]`** — Point it at a `jmp rsi` gadget with 2 args.
2. **Patch `ucred`** — Scan the `proc` struct for the `0x726f7272` ("rror") marker, then walk back to find `ucred`. Set UID/GID to 0 and swap the prison pointer.
3. **Swap root vnode** — Read `rootvnode` and point the file descriptor's `fd_rdir` / `fd_jdir` at it.
4. **Restore `sysent[661]`** — Put the original handler back.

The `read64` / `write64` helpers temporarily retarget `globalFakeArray`'s vector at an arbitrary address, perform the access, and restore the vector.

---

## Supported Firmware

The `PS4_OFFSETS` table covers these firmware versions:

| Range | Versions |
|-------|----------|
| 5.x | 5.00, 5.03, 5.50, 5.53, 5.55, 5.56 |
| 6.x | 6.00, 6.20, 6.50, 6.70 |
| 7.x | 7.00, 7.50 |
| 8.x | 8.00, 8.50 |
| 9.x | 9.00, 9.03, 9.50 |
| 10.x | 10.00, 10.50 |
| 11.x | 11.00, 11.02, 11.50 |
| 12.x | 12.00, 12.50 |
| 13.x | 13.00, 13.02, 13.04, 13.50, 13.52 |

Each entry provides `evf`, `prison0`, `rootvnode`, `sysent`, and `jmp` offsets. Firmware outside this table will fall back to an empty offset set and the jailbreak stage will misbehave — the R/W primitive itself is firmware-agnostic.

The firmware label is derived from the user agent:

```
PlayStation 4/9.00          → "9.00"
PS4 Update (9.00)           → "9.00"
```

---

## Troubleshooting

| Symptom | Likely cause | What to do |
|---------|--------------|------------|
| Status stuck on `RUNNING` | Capture or composition task never fired | Reload from a fresh boot; the heap is likely fragmented |
| `HEADER-MISMATCH` | Fake cell header was not validated | Reboot the console and try once, cleanly |
| `NORMAL-CLONE-MISS` | The duplicate reference returned the original object | Heap placement missed; reboot and retry |
| `SSV-PLACEMENT-MISS` | Groom did not land in the intended hole | Try a different `?n=` value (e.g. `512`, `1024`) |
| `LOAD-THREW` | Loading `history.state` threw a `TypeError` | Reboot; the session may be tainted |
| `SANDBOX-ESCAPE FAILED-not-found` | `ucred` signature scan did not match | Add `?dbg=1` and inspect the `PROC-DUMP` output |
| `JAILBREAK-ERROR` | Kernel write hit an unmapped address | Verify the firmware offsets match your console |
| Page reloads repeatedly | Host/launcher wrapper is reloading it, **or** you are running the old build | This build halts before `runJailbreak()` when `?jb=1` is absent — confirm the URL |
| Status bar stays grey | Exploit script threw during setup | Check the console for `SETUP-THREW` |

**Golden rule:** if the exploit fails, **power-cycle the console** before trying again. A tainted heap will cause every subsequent attempt to fail the same way.

---

## File Overview

| Region | Purpose |
|--------|---------|
| `<style>` | Dark console theme, status bar, toolbar, log line styling |
| `#statusbar` | Live state indicator with PASS/FAIL/WARN chips |
| `#toolbar` | Log export buttons and checkboxes |
| `#consoleWrap` | Scrollable, color-coded console output |
| `LOG` object | In-memory log store, counters, and auto-save state |
| `classify()` | Maps exploit tags to `pass` / `fail` / `warn` / `info` |
| `mark()` | Central logging entry point used by all exploit stages |
| `PS4_OFFSETS` | Per-firmware kernel offsets |
| `RUN_JAILBREAK` | URL-param gate for the kernel stage |
| `buildFakeHost()` | Constructs the fake JSObject with an encoded `ArrayBuffer` header |
| `leakScopeObject()` | Leaks the function scope via a proxy receiver |
| `prepareSymbolWrapper()` | Installs the getter that exposes host/target pointers |
| `loadHistoryCritical()` | Re-acquires the corrupted cell and verifies the fake header |
| `reportComposition()` | Decides PASS / HALTED / FAIL and dispatches the jailbreak gate |
| `read64()` / `write64()` | Arbitrary read/write via the retained fake array |
| `runJailbreak()` | Kernel `sysent` hijack, `ucred` patch, root vnode swap |
| `runAttempt()` | Single-shot entry point |

---

## License & Credits

Exploit author tag: `OptiTronOffical`.
Console UI, logging, and single-shot halt logic: as provided in this file.

Use at your own risk. No warranty is provided, express or implied.
