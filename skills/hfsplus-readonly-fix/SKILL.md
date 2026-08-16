---
name: hfsplus-readonly-fix
description: Use when a Linux machine mounts an HFS+ (macOS) drive read-only, writes fail with "Read-only file system", or kernel logs say "write access to a journaled filesystem is not supported". Also when udisksctl/sudo remounts to rw silently stay read-only.
---

# HFS+ Read-Only Mount Fix

## Overview

Linux refuses to write to **journaled HFS+** (macOS) volumes by design: the kernel driver cannot replay the Mac journal, so it mounts read-only. The fix is to remount with the kernel `force` option. This works without sudo via `udisksctl`.

## When to Use

Use when ALL of these are true:

- Drive is HFS+ (`findmnt` shows `type hfsplus`)
- Mount options show `ro`
- Writing fails: `touch: cannot touch '...': Read-only file system`
- Kernel log confirms the cause

## Detection

```bash
# 1. Confirm the mount is read-only and find its device node
findmnt /media/<user>/<label>          # OPTIONS shows ro, FSTYPE hfsplus
lsblk -f                               # maps label -> device (e.g. sda1)
#   or directly:
findmnt -n -o SOURCE /media/<user>/<label>

# 2. Confirm the kernel's reason
journalctl -b -k --no-pager | grep -i hfsplus
#   hfsplus: write access to a journaled filesystem is not supported,
#   use the force option at your own risk, mounting read-only.
```

If you see that kernel message, a normal `mount -o rw` **will never work** — you must pass `force`.

## Fix (no sudo)

```bash
# 1. Unmount (must be free first — see "Device busy" below)
udisksctl unmount -b /dev/sdXY

# 2. Remount with the force option — THIS is the key flag
udisksctl mount -b /dev/sdXY --options force,rw

# 3. Verify it actually took (OPTIONS must show rw)
findmnt /media/<user>/<label>
```

### If sudo is available (alternative)

```bash
sudo mount -t hfsplus -o force,rw /dev/sdXY /media/<user>/<label>
```

## Device Busy

If unmount fails with `target is busy`, a file manager or process holds it:

```bash
fuser -mv /media/<user>/<label>      # shows holding PIDs
lsof +D /media/<user>/<label>        # shows open files (slow on big drives)
```

Kill the holder (e.g. `nautilus`), then retry the unmount. If the holder reopens the mount, kill it again and immediately unmount; the volume must be free before remounting (`AlreadyMounted`/`DeviceBusy` both mean it is not).

## Common Mistakes

| Mistake | Reality |
|---|---|
| `mount -o rw` silently stays `ro` | Journaled HFS+ needs the `force` option; kernel ignores plain rw |
| Forgetting to unmount first | Remount requires the volume to be free |
| Skipping the busy check | `DeviceBusy` blocks unmount until the holder is gone |
| sudo unavailable | `udisksctl ... --options force,rw` needs no sudo |

## Risks

- The kernel itself warns: **"use the force option at your own risk"**.
- Force-writing a journaled HFS+ volume can corrupt the filesystem or leave it in a state macOS must repair.
- If the volume was not cleanly unmounted, the safest long-term move is to **copy data off** to a Linux-native or exFAT drive, then cleanly eject the HFS+ drive from macOS.
