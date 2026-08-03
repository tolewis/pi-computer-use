# Benchmarks

Measured on real hardware, not estimated. Each row records what was run, where,
and what came back. Re-run these after touching a native bridge.

Fork: `tolewis/pi-computer-use`, forked from `injaneity/pi-computer-use` at
v0.5.0. Recorded 2026-08-02.

## Machines

| Name | OS | Session | Helper |
| --- | --- | --- | --- |
| server | Ubuntu, X11, GNOME | systemd **user** unit in the graphical session | `linux-bridge`, built here |
| jubiku | Windows 11 | scheduled task `/IT`, Session 1 | `windows-bridge.exe`, built here |
| macbook | macOS 15.7.3, x86_64 | LaunchAgent in the Aqua session | `bridge`, prebuilt from git |

A background session on any of these enumerates zero windows and asserts
nothing, which reads as a pass. Every number below is therefore paired with the
session it was taken in.

## Test suites

| Suite | Where | Result |
| --- | --- | --- |
| `cargo test` windows bridge | jubiku, Session 0 | 48 passed, 0 failed |
| `cargo test` windows bridge | jubiku, Session 1 | 48 passed, 0 failed |
| `cargo test` linux bridge | server, X11 | 19 + 11 + 3 passed, 0 failed |

Upstream v0.5.0 did not compile under `cargo test` at all: `capture.rs` called
`screenshot()` with three arguments after a fourth was added.

The windows suite previously passed only from Session 1. `SendInput` needs an
interactive window station, and Windows reports its absence two different ways:
`0x800705B3` for pointer actions, and a silent `inserted 0/N events` for
keyboard actions.

## Root enumeration

`find_roots`, live, non-headless.

| Machine | Roots | Sample |
| --- | --- | --- |
| server | 8 | Teams, Brave, Chrome, gnome-shell |
| jubiku, Session 1 | 25 | WindowsTerminal, Orchestrate, Brave, QuickBooks |
| jubiku, Session 0 | **0** | isolation, not an error |
| macbook | blocked | TCC not yet granted |

## Window activation

The defect class this fork exists to fix. Both backends were wrong, in
opposite directions.

| Backend | Before | After |
| --- | --- | --- |
| Windows | "Windows refused to foreground the target; physical input was not sent" | foreground acquired; a click on the QuickBooks shortcuts panel opened the Customer Center |
| X11 | returned `focused: true` without ever looking | reads back the active window; reports `focused` and `alreadyFocused` truthfully |

Windows failed honestly. X11 succeeded falsely.

Regression coverage: the X11 test targets an unmapped window id, which can
never become active. It was confirmed to fail against the previous
implementation before being accepted.

## Input throughput

A single `act_ui` `typeText` of `ABCDEFGHIJ` delivers all 10 characters in one
call. There is **no** one-character-per-call limit; an earlier report of one was
a misdiagnosis. `setText` writes a whole value at once through the UIA
ValuePattern and is the faster path for replacing field contents.

## Build times

Release builds, warm cargo cache.

| Target | Machine | Time |
| --- | --- | --- |
| windows-bridge | jubiku | 24 s |
| linux-bridge | server | 9 s incremental, 35 s cold |

Only `prebuilt/macos` is committed to git, so Linux and Windows hosts build
their own helper. `setup-helper.mjs` now does that automatically when cargo is
on PATH.
