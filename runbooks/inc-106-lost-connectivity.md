# INC-106 — Lost Monitoring Connectivity (Capstone)

**Affected host:** inc-target (192.168.122.100)
**Severity:** Average (Zabbix classification)
**Detection method:** Automated — Zabbix agent-availability trigger
**Status:** Resolved and verified

## Summary

This capstone scenario deliberately stacked **two independent causes at
once** behind the same generic "agent not available" symptom already seen
in INC-103/104 — a closed firewall port, *and* the agent's own `ListenIP`
restricted to loopback only. Fixing either alone would not have resolved
the incident; both had to be identified and cleared, one at a time, with
the target host's actual state checked directly rather than inferred
from client-side symptoms.

## Symptoms

- Zabbix: `Linux: Zabbix agent is not available (for 3m)` — Average —
  same generic trigger/wording as INC-103 and INC-104
- `zabbix_get` from `zbx-server` returned:
  `connection error (POLLERR,POLLHUP)`
- SSH access to `inc-target` remained fully functional throughout

## Root cause (two independent, stacked causes)

1. `10050/tcp` removed from `inc-target`'s active firewalld zone (`public`)
2. The agent's `ListenIP` restricted to `127.0.0.1` only (default is all
   interfaces) — confirmed via `ss -tlnp` showing the process bound to
   loopback specifically, not `0.0.0.0`

## Diagnosis — and a real, worth-documenting mistake along the way

The initial working theory was that a firewall block and a
"nothing listening" condition would produce visibly *different*
`zabbix_get` error text — reasoning that a firewall reject typically
shows as an immediate network-level refusal, versus a clean OS-level
"connection refused" when nothing's bound to that address.

**That theory turned out to be wrong in this case.** Reopening the
firewall alone (confirmed via `firewall-cmd --list-ports` genuinely
showing `10050/tcp` restored) produced the *exact same*
`POLLERR,POLLHUP` error as before, with `ListenIP` still restricting the
bind. Zabbix's own `zabbix_get` reports any immediately-reset TCP
connection the same way, regardless of whether the reset came from a
firewall rule or from nothing being bound at that address — the
client-side error text does not reliably distinguish the two causes here.

**The actual, more valuable lesson**: don't infer root cause from how a
connection attempt fails on the client side — verify the target's real
state directly instead:

```bash
sudo ss -tlnp | grep 10050
```

This showed `LISTEN 127.0.0.1:10050` — direct, unambiguous proof the bind
restriction was still active, independent of anything `zabbix_get`
reported.

## Resolution

Fixed one layer at a time, deliberately, confirming state directly after
each step rather than trusting the client symptom to improve on its own:

**Layer 1 — firewall:**
```bash
sudo firewall-cmd --permanent --add-port=10050/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports
```
Confirmed genuinely reopened via firewalld's own state query — but
`zabbix_get` still failed identically, proving Layer 2 was independently
still in effect (see diagnosis above).

**Layer 2 — agent bind restriction:**
```bash
sudo sed -i -E 's/^ListenIP=.*/# ListenIP=0.0.0.0/' /etc/zabbix/zabbix_agent2.conf
sudo systemctl restart zabbix_agent2
sudo ss -tlnp | grep 10050
```

## Verification

- `ss -tlnp` confirmed the listener moved from `127.0.0.1:10050` to
  `*:10050` (all interfaces) after the second fix
- `zabbix_get -s 192.168.122.100 -p 10050 -k agent.ping` returned `1`
- Zabbix: problem transitioned to **RESOLVED**; Problems view returned to
  "No data found"

## Notes for reproducibility / lessons learned

- The identical generic "agent not available" alert was deliberately
  reused a third time (after INC-103 and INC-104) to reinforce, at
  capstone scale, that the same alert text can hide entirely different —
  and here, multiple *simultaneous* — root causes.
- A real, corrected mistake worth keeping visible rather than editing
  out: an initial hypothesis about client-error-message specificity was
  tested and found wrong. The fix was falling back on direct
  target-state verification (`ss -tlnp`) rather than continuing to infer
  from symptom text — the same "verify, don't guess" discipline used
  throughout this entire project.
- When troubleshooting connectivity, checking a service's actual bind
  address (`ss -tlnp`) and the firewall's actual current rule state
  (`firewall-cmd --list-all`, or deeper still, `nft list ruleset`)
  independently is more reliable than reasoning backward from how a
  remote client's connection attempt failed.
- Unrelated finding, logged for awareness only: `firewall-cmd
  --get-active-zones` revealed a `docker` zone bound to a `docker0`
  interface on `inc-target` — Docker is installed and running here,
  unrelated to anything deliberately set up as part of this project. Not
  investigated further; noted in `docs/gotchas.md`.
