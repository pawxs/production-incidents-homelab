# INC-101 — Disk Space Exhaustion

**Affected host:** inc-target (192.168.122.100)
**Severity:** Average (Zabbix classification)
**Detection method:** Automated — Zabbix, "Linux by Zabbix agent" template
**Status:** Resolved and verified

## Summary

A runaway/unrotated log file filled the shared Btrfs storage pool underlying
every subvolume on `inc-target`, driving disk usage to 91%. Because all
subvolumes on this host share one physical pool rather than having
independent capacity, **10 separate filesystem triggers fired
simultaneously** — one for every discovered mount point, not just the one
that "looks" full.

## Symptoms

Zabbix raised 10 concurrent problems, all at the same timestamp
(`2026-09-19 03:09:34 PM`), all reading:

```
Linux: FS [<mountpoint>]: Space is critically low (used > 90%, total 18.0GB)
```

Affected mountpoints: `/`, `/home`, `/opt`, `/root`, `/srv`, `/usr/local`,
`/.snapshots`, `/boot/grub2/i386-pc`, `/boot/grub2/x86_64-efi`, `/var`.

## Root cause

`inc-target` uses Btrfs with multiple subvolumes (`@`, `@/home`, `@/var`,
etc.) mounted at different paths — but all subvolumes draw from **one
shared underlying block-group pool**, not independent partitions. A single
oversized file anywhere in that pool reduces free space for *every*
subvolume identically, so a threshold breach on one path is a threshold
breach on all of them at once. This is a meaningful behavioral difference
from traditional partitioned filesystems (ext4/XFS on LVM), where filling
`/var` has no effect on `/home`'s reported free space.

In this case, the oversized file was `/var/log/inc101-runaway.log`,
created deliberately as a controlled simulation of an application log
growing unbounded.

## Diagnosis

Confirm actual usage rather than trusting the alert text alone:

```bash
df -h /
sudo btrfs filesystem usage /
```

Find the offending file(s):

```bash
sudo du -xh /var/log 2>/dev/null | sort -rh | head -10
```

**Gotcha worth knowing:** `du`'s per-directory breakdown does not list a
large file as its own line if that file sits *directly* inside the
directory being scanned — its size is folded into the parent directory's
own total instead. In this incident, `/var/log` showed as `14G` while
every named subdirectory beneath it summed to under 100MB. When `du`'s
total and its listed children don't add up, check `ls -la` directly on
the suspect directory — the largest file is often sitting loose at that
level, not hidden in a subfolder.

## Resolution

```bash
sudo rm /var/log/inc101-runaway.log
df -h /
```

## Verification

- `df -h /` returned to baseline: **14% used** (down from 91%)
- All 10 Zabbix problems transitioned to **RESOLVED** automatically within
  one polling interval, with no manual acknowledgement needed to clear them
- Recovery duration: 22h 12m from detection to resolution (test scenario —
  the delay reflects when the fix was applied, not detection latency,
  which was under Zabbix's normal polling interval)

## Notes for reproducibility

- The test file was created with `fallocate -l 14G` (fast, allocates real
  extents without writing actual data byte-by-byte — Btrfs and `df` count
  this identically to genuinely written data).
- A duplicate `dd if=/dev/zero ... bs=1M count=14000` was run against the
  same path afterward (a filename typo caused a false "file doesn't exist"
  read). GNU `dd` truncates its output file to zero length by default
  before writing, unless `conv=notrunc` is passed — so this replaced
  rather than stacked on top of the fallocated file, landing at the same
  intended ~91% target rather than genuinely exhausting the disk.
- Zabbix's WARNING/CRITICAL thresholds for this template default to 80%/90%
  used — not customized on this host.
