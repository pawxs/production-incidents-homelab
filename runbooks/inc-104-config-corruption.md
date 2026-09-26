# INC-104 — Configuration Corruption (Snapper Rollback)

**Affected host:** inc-target (192.168.122.100)
**Severity:** Average (Zabbix classification)
**Detection method:** Automated — Zabbix agent-availability trigger
(same trigger as INC-103)
**Status:** Resolved and verified

## Summary

The Zabbix Agent 2 configuration file on `inc-target` was deliberately
corrupted (a single parameter changed to an invalid value) to simulate a
bad config change breaking a critical service. Unlike INC-103, the fix
here is not a restart or reversing a service state — it's a genuine
Snapper `undochange`, reverting the file to its exact pre-corruption
content.

`zabbix_agent2` was deliberately reused as the target service (same as
INC-103) rather than `sshd`, for two reasons: avoiding any risk of losing
remote access mid-test, and to make a point explicit — the Zabbix-side
symptom ("agent not available") is **identical** to INC-103's. A real
monitoring alert usually can't tell you *why* something is down, only
*that* it is; the actual diagnosis has to come from the host itself, not
the alert text.

## Symptoms

- Zabbix: `Linux: Zabbix agent is not available (for 3m)` — Average —
  same trigger, same wording as INC-103
- SSH access to `inc-target` remained fully functional throughout

## Root cause

`/etc/zabbix/zabbix_agent2.conf` line 83, `ListenPort`, was changed from
its commented-out default to an invalid literal value
(`ListenPort=notanumber`) — a stand-in for a real-world fat-fingered
config edit.

## Diagnosis

Because the symptom alone is indistinguishable from INC-103, the journal
is what actually differentiates a config problem from a stopped or
crashed process:

```bash
sudo systemctl status zabbix_agent2 --no-pager
sudo journalctl -u zabbix_agent2 --no-pager | tail -20
```

The service's own startup code caught the invalid value immediately and
reported it precisely:

```
ERROR: Cannot assign configuration: invalid parameter ListenPort at line 83: strconv.ParseInt: parsing "notanumber": invalid syntax
```

No ambiguity — the exact file, line number, and bad value are all named
directly. This is the actual differentiator from INC-103: a crash or stop
produces a generic process-exit story; a config error like this names the
specific problem outright.

## A genuine systemd gotcha, worth documenting

The service auto-restarted every ~10 seconds (`RestartUSec=10s`,
confirmed in INC-103) for hours straight — the journal showed a restart
counter climbing past 180 attempts overnight, never settling into a
permanent `failed` state despite `StartLimitBurst=5` being configured.
Reason: `StartLimitBurst` only trips if that many failures occur *within
a single rate-limiting interval*, and this unit's ~10-second restart
spacing happens to roughly match its own rate-limit window, so the
failure count essentially never accumulates past one-per-window before
the window resets.

**Lesson:** `Restart=` plus `StartLimitBurst=` does not automatically
guarantee a crash-looping service will eventually give up and enter
`failed` — the real protection depends on how the restart delay compares
to the rate-limit interval, and shouldn't be assumed without checking.

## Resolution — Snapper rollback (the actual fix for this incident type)

```bash
# Snapshot before the change (in a real incident, this would already
# exist as a routine pre-change snapshot, not created reactively)
PRE=$(sudo snapper create --type single --print-number --description "pre inc-104 config corruption")

# ... corruption happens ...

# Snapshot after the change
POST=$(sudo snapper create --type single --print-number --description "post inc-104 config corruption")

# Confirm what actually changed between the two
sudo snapper status ${PRE}..${POST}

# Revert precisely that change
sudo snapper undochange ${PRE}..${POST}

sudo systemctl restart zabbix_agent2
```

`snapper status` correctly reported a **content-modification** marker
(`c.....`) for the affected file — a meaningfully different signature
from INC-101's Snapper test, which only ever demonstrated a file
*creation* (`+.....`). This confirms Snapper correctly tracks in-place
edits to existing files, not just additions or deletions. `undochange`
reported `create:0 modify:1 delete:0` — exactly one file modified,
nothing else touched — and a direct `grep` afterward confirmed the file
was restored to its exact original content (`# ListenPort=10050`, back
to commented-out), not merely replaced with some other valid value.

## Verification

- `systemctl status` returned to `active (running)` immediately after the
  restart, clean startup log, no errors
- Zabbix: problem transitioned to **RESOLVED**; host availability
  returned to **2/2 Available**

## Notes for reproducibility / lessons learned

- The same target service as an earlier scenario was reused deliberately,
  to demonstrate that identical-looking alerts can have entirely
  different root causes and require entirely different fixes — alert
  text alone is never sufficient for diagnosis.
- `snapper status <pre>..<post>` distinguishes create (`+`), delete
  (`-`), and content-modify (`c`) changes — worth knowing which marker to
  expect depending on the type of change being investigated.
- See the systemd gotcha above regarding `StartLimitBurst` not reliably
  preventing an indefinite crash-loop when restart timing and the
  rate-limit window happen to align.
