# INC-103 — Critical Service Down (Zabbix Agent)

**Affected host:** inc-target (192.168.122.100)
**Severity:** Average (Zabbix classification)
**Detection method:** Automated — Zabbix built-in agent-availability trigger
**Status:** Resolved and verified

## Summary

The Zabbix Agent 2 service on `inc-target` was deliberately brought down to
simulate a critical monitoring service failing. The first attempt
(`SIGKILL`, simulating a crash) was healed automatically by systemd's own
`Restart=on-failure` policy within 10 seconds — faster than Zabbix's
polling interval could register the gap, so no problem ever appeared. The
scenario was redesigned around a clean `systemctl stop` instead, which
systemd does **not** auto-restart, producing a genuine sustained outage.

## Symptoms

- Zabbix: host availability dropped to **1/2 Available**
- Trigger fired: `Linux: Zabbix agent is not available (for 3m)` — Average
- SSH access to `inc-target` remained fully functional throughout — only
  the agent process was affected, not the host itself, exactly as a real
  monitoring-agent failure would look

## Root cause

`zabbix_agent2.service` was stopped via `systemctl stop` — a deliberate,
clean shutdown (SIGTERM, clean exit), as opposed to a crash.

## Diagnosis

Starting from only "agent unavailable" as the known signal, as a real
ticket would:

```bash
sudo systemctl status zabbix_agent2 --no-pager
sudo journalctl -u zabbix_agent2 --no-pager | tail -20
```

**The key diagnostic signal is telling a clean stop apart from a crash:**

- **Clean stop:** `Stopping...` → `<service> stopped` →
  `Deactivated successfully` → `Stopped <service>`, with
  `ExecStop=... (code=exited, status=0/SUCCESS)` and **no**
  "Scheduled restart job" line anywhere.
- **Crash** (see notes below): `Failed with result 'signal'` immediately
  followed by `Scheduled restart job, restart counter is at N`.

That distinction — present or absent restart-scheduling — is how you tell
"this crashed" from "someone or something stopped this on purpose"
directly from the journal, without any other context.

## Resolution

```bash
sudo systemctl enable --now zabbix_agent2
sudo systemctl status zabbix_agent2 --no-pager
```

(`enable --now`, not just `start` — restores it to surviving a future
reboot, matching how it was originally deployed.)

## Verification

- `systemctl status` showed `active (running)` with a fresh start timestamp
- Zabbix: the problem transitioned to **RESOLVED**; host availability
  returned to **2/2 Available**

## Notes for reproducibility / lessons learned

- The first attempt used `sudo systemctl kill -s SIGKILL zabbix_agent2` to
  simulate a crash. This genuinely worked — confirmed via
  `Failed with result 'signal'`, `code=killed, signal=KILL` — but the
  unit's own `Restart=on-failure` policy (`RestartUSec=10s`, confirmed via
  `systemctl show`) auto-healed it within 10 seconds, well inside a single
  Zabbix polling interval, so no problem was ever visible in the
  dashboard. Worth keeping in mind as its own valid finding: a resilient
  systemd unit can recover faster than monitoring can catch the gap.
- Redesigned around `systemctl stop` because `Restart=on-failure` only
  triggers on an abnormal (non-zero/signal) exit — a deliberate stop is
  never auto-restarted. Arguably more realistic besides: "found stopped,
  cause unknown" is one of the most common real service-down ticket types.
- `StartLimitBurst=5` was also confirmed present on this unit (systemd
  would eventually give up and enter a permanent `failed` state after 5
  crash-restarts within its configured interval). Not used for this
  scenario, but a legitimate mechanism for a harder future variant of the
  same incident type.
