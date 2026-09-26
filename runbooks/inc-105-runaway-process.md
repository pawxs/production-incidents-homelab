# INC-105 — Runaway Process / High CPU Load

**Affected host:** inc-target (192.168.122.100)
**Severity:** Warning (Zabbix classification)
**Detection method:** Automated — Zabbix CPU utilization trigger
**Status:** Resolved and verified

## Summary

Two CPU-bound busy-loop processes (`yes > /dev/null`, one per vCPU) were
deliberately launched on `inc-target` to simulate a runaway process
pegging the CPU. Zabbix's "High CPU utilization" trigger fired only after
usage stayed above 90% for its full configured evaluation window,
confirming the trigger is designed to react to sustained load, not an
instantaneous spike.

## Symptoms

- Zabbix: `Linux: High CPU utilization (over 90% for 5m)` — Warning
- `inc-target` became noticeably less responsive over SSH during the load
  period — expected, given both of its 2 vCPUs were fully saturated
- `uptime`'s load average climbed from a near-zero baseline toward ~2.0
  (matching two fully-loaded cores) over roughly a minute — load average
  is a smoothed, minute-scale figure, not instantaneous, so it visibly
  lags behind real-time CPU%.

## Root cause

Two processes named `yes` (PIDs 8587 and 8588), each consuming ~99.8% of
one core continuously.

## Diagnosis

Starting from only the Zabbix alert, as a real ticket would:

```bash
ps aux --sort=-%cpu | head -10
```

Immediately surfaces the offending PIDs at the top, sorted by CPU
descending — the standard first move for any CPU-pegged host.
`top -bn1 | head -20` is an equally valid interactive-style alternative.

## Resolution

```bash
kill <pid1> <pid2>
```

Using the actual PIDs identified by `ps`/`top` — not a name-based
`pkill` — confirms you're stopping the specific process you diagnosed,
not merely one that happens to share a name. More defensible in a real
incident, where guessing by name risks killing the wrong instance.

## Verification

- Shell confirmed both jobs `Terminated`
- `uptime`'s load average began descending immediately: the 1-minute
  figure had already dropped to 1.38 within 5 seconds of the kill, while
  the 5-minute figure (1.86) still reflected the recent spike — the three
  windows update asynchronously, so a fresh drop shows up first in the
  shortest one.
- Zabbix: problem transitioned to **RESOLVED** at 10:10:14 AM — 14 minutes
  total duration, and essentially real-time with when the fix was applied.

## Notes for reproducibility / lessons learned

- `yes > /dev/null`, one instance per `nproc` core, is a simple,
  dependency-free way to saturate CPU on a Linux host for testing — no
  external tool (e.g. `stress-ng`) required.
- The trigger's `(over 90% for 5m)` condition is a deliberate design
  choice, not a delay or a bug — it exists specifically to avoid flapping
  alerts on legitimate short CPU bursts (a backup job, a compile),
  escalating only once high usage is clearly sustained.
- This load-generation method is essentially RAM-free (~4MB RSS per
  process), so it stresses CPU in isolation without compounding any
  existing host-side memory pressure — a useful property on a
  resource-constrained physical host like this one.
