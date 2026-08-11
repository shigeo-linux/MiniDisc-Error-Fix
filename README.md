# MiniDisc "Error" Fix

How to recover a Sony NetMD MiniDisc that shows **"Error"** on the deck's display
instead of "Blank" when inserted, using raw NetMD protocol commands sent via
[`netmdcli`](https://github.com/linux-minidisc/linux-minidisc) — no factory
service mode required.

Tested on a Sony NetMD Walkman MZ-N505 (USB ID `054c:0084`) under Ubuntu Linux 24.04.

## Prerequisites

Sony NetMD Walkman deck; PC with Ubuntu 24.04 installed; MiniDisc that fails to read,
USB cable USB-A to Mini-USB (Mini-B, 5-pin) standard connector, not a
proprietary Sony one.  

`netmdcli` installed from https://github.com/linux-minidisc/linux-minidisc.

For netmdcli on Ubuntu, there's no traditional kernel driver needed — NetMD
devices don't have a generic USB class (not mass storage, not audio class), so
nothing binds to them at the kernel level, and netmdcli talks to the device
directly via libusb from userspace. 

Runtime packages:
`sudo apt install libusb-1.0-0 libgcrypt20`
(confirmed via ldd on the built binary — it also links libudev, libgpg-error,
libcap, but those are already present on any stock Ubuntu install)

Build-time packages (only if compiling from source, like this install was):
`sudo apt install build-essential libusb-1.0-0-dev libgcrypt20-dev qtbase5-dev
pkg-config`

udev permission rule (this is the part that actually matters — without it you
need sudo for every netmdcli command, since raw USB device nodes are root-only
by default). The test machine already has one installed at
/etc/udev/rules.d/50-netmd.rules covering every known NetMD/Hi-MD device ID,
plus the user in the plugdev group. If you're setting this up on a different
Ubuntu machine, the relevant line for specific deck (MZ-N505) is:

`ATTRS{idVendor}=="054c", ATTRS{idProduct}=="0084", MODE="0664", GROUP="plugdev"`

worth noting: 0084 identifies the Sony MZ-N505


## Symptom

A disc shows `Error` (sometimes `00:00 Error`) on the deck's own screen when
inserted, instead of `Blank` or a track title. `netmdcli` (run with no
arguments) reads the disc as `<Untitled>` with 0 tracks and no groups — the
same as a genuinely blank disc.

This is a **corrupted logical TOC** (table of contents), not:

- The physical write-protect tab (that only blocks recording; it doesn't
  affect whether the deck can read/mount the TOC at all).
- A Hi-MD/MD-DATA disc in a non-Hi-MD deck (USB communication works fine in
  that case too, it just returns a nonsensical TOC — the symptom looks
  similar, but that's a different underlying cause and this fix won't help).

Also a dead end: the Sony factory **"Test Mode"** documented in official
service manuals is a laser/servo/CD-MO **calibration** mode. It has nothing to
do with formatting discs — don't waste time chasing that path for this
problem.

## The fix

`netmdcli` has no built-in erase/format command (checked as of the current
`linux-minidisc/linux-minidisc` upstream and the `glaubitz` fork — neither
has one). Instead, send the raw NetMD protocol bytes directly:

```sh
netmdcli raw 0018081018020300   # cache TOC
netmdcli raw 001840ff0000       # erase
netmdcli raw 0018081018020000   # sync TOC (commits the erase to the physical disc)
```

Run all three back-to-back, **without ejecting the disc in between** — the
sync step is what actually commits the change; skipping it (or ejecting
before it runs) discards the erase.

These bytes were cross-checked two ways before use: they structurally match
the `eraseDisc()` / `cacheTOC()` / `syncTOC()` raw queries in a separate
Python NetMD implementation, and the cache/sync byte patterns are identical
to what this repo's own C `libnetmd` already sends for `netmd_cache_toc()` /
`netmd_sync_toc()` (`libnetmd/libnetmd.c`) for other operations — same
protocol, just a different opcode for erase.

After running the sequence, **physically eject and reinsert the disc** — the
deck's display doesn't refresh live, so you won't see the result until you
do.

## Reading the outcome

On reinsert, the deck shows `Edit` while it writes the new TOC to the disc.

- **Success:** clears to `Blank` within well under a minute.
- **Failure:** `Edit` persists — observed anywhere from 6 minutes to 76+
  minutes, with the deck still audibly seeking/writing (not a frozen
  display — an actual ongoing retry loop). Pressing Stop typically does
  nothing (may flash `PC->MD` briefly, then right back to `Edit`).

If it's still on `Edit` past roughly 5-10 minutes, it's not going to
complete — that's not a slow legitimate write, it's the deck retrying a
failed write against what's almost certainly **physical damage** at the
TOC-write area of the disc. No amount of additional waiting or re-running
the erase sequence fixes that; those discs are done.

In one batch of 5 discs tested this way: 2 recovered cleanly, 3 failed with
this pattern and are presumed physically damaged.

### Recovering the deck itself from a stuck "Edit"

1. Remove the battery (don't wait for the battery to die on its own — a
   power-cycle you control is safer than one that happens mid-write).
2. Wait ~10 seconds, reinsert the battery, power on **with no disc** first.
3. Confirm it shows `NoDisc` — that means the deck itself is fine.
4. Reinsert the disc. It'll show `Error` again — i.e. still unrecovered, but
   the deck is healthy and ready for the next disc.

## Known gotchas

- **`netmdcli capacity` is unreliable on at least this deck model** — it
  queries the device directly (`netmd_get_disc_capacity` /
  `libnetmd/playercontrol.c`), but reports a bogus near-zero total
  (`00:00:00.95`) even for discs the deck itself correctly shows as `Blank`
  and reads with a full, correct track listing. Don't use it to judge
  whether a disc is actually fixed — trust the deck's own front-panel
  display and/or `netmdcli`'s track/group listing instead.
- **USB connection is flaky during handling.** Expect frequent
  `usb 1-x: USB disconnect` events (visible via `journalctl -k`) whenever
  the disc is swapped or the deck is disconnected to check its native
  display. A reseat/wiggle of the connector on the deck side usually brings
  it back. This seems to be a general trait of these old connectors, not
  specific to one faulty unit.
