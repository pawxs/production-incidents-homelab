# Production Incidents Homelab

A self-built, fully documented homelab simulating real Linux production
incidents — each one actually broken, diagnosed from scratch, fixed, and
verified recovered, with every step backed by real command output rather
than assumed.

## What this is

Two KVM virtual machines running openSUSE Leap 16.0: one hosting a full
Zabbix 7.0 monitoring stack (server, MariaDB, Apache/PHP frontend), the
other a monitored target hardened with fail2ban and Snapper/Btrfs
snapshotting. Six incident scenarios were deliberately engineered against
this environment — each diagnosed as if the cause were unknown, using
only what a monitoring alert and the host itself reveal, then fixed and
confirmed recovered in the dashboard.

## Architecture

| Host | Role | OS |
|---|---|---|
| `zbx-server` | Zabbix 7.0.28 server, MariaDB, Apache/PHP frontend | openSUSE Leap 16.0 |
| `inc-target` | Monitored host — Zabbix Agent 2, fail2ban, Snapper/Btrfs | openSUSE Leap 16.0 |

Both run under KVM/libvirt on a single physical Linux machine, on a
private NAT'd network.

## Skills demonstrated

Linux system administration (openSUSE/SUSE family) · KVM/libvirt
virtualization · Zabbix deployment and custom monitoring integrations ·
fail2ban and firewalld/SELinux hardening · Btrfs/Snapper backup and
recovery · systemd/journald-based troubleshooting · incident
documentation and runbook writing.

## Incident runbooks

Each incident was deliberately engineered, diagnosed as if the cause
were unknown, fixed, and verified recovered in the monitoring dashboard —
not written up after the fact from memory.

| # | Incident | Core skill demonstrated |
|---|---|---|
| [inc-101](runbooks/inc-101-disk-space-exhaustion.md) | Disk space exhaustion | Filesystem triage; Btrfs shared-pool behavior |
| [inc-103](runbooks/inc-103-critical-service-down.md) | Critical service down | `systemctl`/`journalctl` diagnosis; crash vs. clean-stop |
| [inc-105](runbooks/inc-105-runaway-process.md) | Runaway process / high CPU load | Process triage; `ps`/`top`; load-average interpretation |
| [inc-102](runbooks/inc-102-ssh-brute-force.md) | SSH brute-force attempt | Security monitoring; building custom Zabbix telemetry for fail2ban |
| [inc-104](runbooks/inc-104-config-corruption.md) | Configuration corruption | Snapper rollback as the actual fix, not a manual edit |
| [inc-106](runbooks/inc-106-lost-connectivity.md) | Lost monitoring connectivity (capstone) | Multi-layer network/firewall diagnosis; two independent stacked causes |

## Other documentation

[`docs/gotchas.md`](docs/gotchas.md) — every real deviation hit along the
way: openSUSE-specific Zabbix packaging quirks, SELinux fixes, fail2ban
and firewalld integration, and a few process lessons learned the hard
way — kept for reproducibility, not polished away.

## Why this project

Built as hands-on practice for a transition into Linux system
administration and NOC/IT support work — treating each incident the way
an on-call engineer actually would: starting from only what an alert
shows, diagnosing from the system itself, and documenting the real fix
afterward.

## Honest limitations

- A two-VM homelab on a single physical machine, not a production
  environment — some realism trade-offs are called out explicitly where
  they matter (e.g. inc-102's "attacker" address is the trusted host
  machine, not a genuinely external one).
- Mistakes made along the way are left visible in the runbooks and
  gotchas doc rather than edited out — including, in inc-106, a wrong
  working hypothesis that was tested and corrected based on evidence.
