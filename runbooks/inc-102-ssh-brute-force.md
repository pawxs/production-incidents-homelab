# INC-102 — SSH Brute-Force Attempt

**Affected host:** inc-target (192.168.122.100)
**Severity:** Average (Zabbix classification)
**Detection method:** Automated — custom Zabbix item exposing fail2ban's
live ban state (built as part of this incident; did not previously exist)
**Status:** Resolved and verified

## Summary

A simulated SSH brute-force attack (3 rapid wrong-password attempts)
against `inc-target` triggered its existing sshd fail2ban jail, which
banned the source IP within seconds. Zabbix had no visibility into
fail2ban at all going into this incident — building that visibility was
part of the work. This is also the first scenario in the set where
detection *and* containment both happen automatically, before any human
intervenes; the actual on-call task is confirming containment worked, not
fixing anything.

## Symptoms

- Zabbix: `fail2ban has banned an IP on inc-target: 1` — Average
- `fail2ban-client status sshd` independently confirmed the same fact:
  `Currently banned: 1`, `Banned IP list: 192.168.122.1`
- SSH connections from the banned address were actively refused for the
  full 10-minute `bantime` window

## Root cause

3 SSH connection attempts with an invalid username/password from the same
source IP within fail2ban's `findtime` window, exceeding the jail's
configured `maxretry = 3`.

## Building the detection pipeline (prerequisite work)

Zabbix's default "Linux by Zabbix agent" template has no fail2ban
awareness — this had to be built first:

1. Confirmed fail2ban's control socket is root-only (`srwx------`),
   settling that a scoped `sudoers.d` rule was the correct approach —
   not a group-membership workaround.
2. Created a dedicated `Include` directory for local custom checks,
   deliberately separate from the vendor-shipped `plugins.d/` (which
   holds native Go plugin configs — MySQL, Redis, etc. — not shell
   `UserParameter`s):
   ```bash
   sudo mkdir -p /etc/zabbix/zabbix_agent2.d/userparams.d
   sudo sed -i '\#^Include=./zabbix_agent2.d/plugins.d/\*.conf#a Include=./zabbix_agent2.d/userparams.d/*.conf' /etc/zabbix/zabbix_agent2.conf
   ```
3. Added the `UserParameter`:
   ```
   UserParameter=fail2ban.sshd.banned,sudo /usr/bin/fail2ban-client status sshd | awk -F: '/Currently banned/{gsub(/[ \t]+/,"",$2); print $2}'
   ```
4. Granted the minimum possible sudo scope — one exact command, nothing
   broader:
   ```
   zabbix ALL=(root) NOPASSWD: /usr/bin/fail2ban-client status sshd
   ```
5. Verified as the actual `zabbix` user directly
   (`sudo -u zabbix sudo -n ...`), then confirmed via `zabbix_get` from
   `zbx-server` over the real agent protocol — both before touching the
   web UI at all.
6. Created the Item (`fail2ban.sshd.banned`, Numeric unsigned, 30s
   interval) and Trigger (`last(/inc-target/fail2ban.sshd.banned)>0`)
   under Data collection → Hosts → inc-target.

## Simulating the attack

```bash
for i in 1 2 3; do
  sshpass -p "wrongpassword" ssh -o StrictHostKeyChecking=no -o PreferredAuthentications=password baduser@192.168.122.100
done
```

Scripted deliberately, not typed manually — an earlier attempt elsewhere
in this project failed to trigger a ban because manual typing took long
enough to hit sshd's `LoginGraceTime` (120s) before all 3 attempts
completed. Scripting removes that timing risk entirely.

## Diagnosis / confirmation

Two independent sources, deliberately cross-checked rather than trusting
either alone:

```bash
# On inc-target
sudo fail2ban-client status sshd
```

```bash
# From zbx-server — the real monitoring path
zabbix_get -s 192.168.122.100 -p 10050 -k fail2ban.sshd.banned
```

## Resolution

None required — fail2ban's own action banned the offending IP
automatically, before any human intervened. The task here is
confirmation, not remediation.

## Verification

- Ban confirmed in `/var/log/fail2ban.log`:
  `2026-09-23 22:45:43,102 ... Ban 192.168.122.1`
- Auto-expiry confirmed at
  `2026-09-23 22:55:43,118 ... Unban 192.168.122.1` — **exactly 600
  seconds later**, matching the configured `bantime` precisely
- Zabbix: the problem cleared on its own with the ban's expiry, no
  manual acknowledgement needed — Problems view returned to "No data
  found"

## Notes for reproducibility / lessons learned

- **Honest limitation, stated plainly**: this lab has only one external
  vantage point — the host machine, whose IP (`192.168.122.1`) is also
  inherently trusted (it's the gateway for this network). A real
  brute-force attack would originate from a genuinely external, untrusted
  address. The detection and containment mechanism is fully real and
  correctly demonstrated; only the attacker's origin is a simulated
  stand-in.
- Not every incident type resolves the same way: this is the first
  scenario where detection and remediation are the same automated action,
  completed before a human ever sees the alert. The skill demonstrated is
  building the missing visibility and confirming containment — not
  diagnosing and fixing.
- Keeping local custom checks in their own `Include` directory
  (`userparams.d/`), separate from vendor-shipped `plugins.d/`, is a
  worthwhile habit on its own — it keeps vendor configs untouched and
  avoids future confusion about which files are safe to hand-edit.
