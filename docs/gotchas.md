# Deviations & Gotchas — production-incidents-homelab

Permanent reference. Unlike `PROJECT_STATE.md`, this file is not overwritten
each session — it only grows, as a knowledge base for anything that differed
from a naive/default setup.

## Host & hypervisor (Fedora 44 Workstation, T490)

- KVM/QEMU via libvirt, confirmed bare metal via `virt-what`.
- Non-root `virsh`/`virt-install` defaulted to `qemu:///session` instead of
  `qemu:///system` — fixed with `LIBVIRT_DEFAULT_URI=qemu:///system` in
  `~/.zshrc`.
- A `distrobox` setup for an unrelated project later disrupted `.zshrc`
  (lost the URI line; new terminals briefly opened in bash instead of zsh
  due to a leaked `$SHELL` env value) — line restored, an accidental
  double-append deduped.
- Pre-existing "Downloads" storage pool under `qemu:///system`, likely from
  GNOME Boxes — left alone, unused by this project.

## VM build (openSUSE Leap 16.0)

- Guest OS: Leap 16.0 (stable), not Tumbleweed or 16.1 (Beta at build time).
- Topology: `zbx-server` + `inc-target`, 2 vCPU / 2048MB (max 3072MB) /
  20GB disk each.
- The `qemu` user couldn't traverse `/home/pawxs` to read the install ISO —
  moved it into `/var/lib/libvirt/images` instead.
- Leap 16 requires an x86-64-v2 CPU baseline; QEMU's default virtual CPU
  lacks it, causing a boot kernel panic — fixed with `--cpu host-passthrough`.
- The Agama installer sanitizes usernames (strips hyphens) — `zbx-server1`
  became login `zbxserver1`, causing real SSH login confusion on the first
  build. Both VMs were ultimately rebuilt with hyphen-free usernames
  matching their hostnames (`zbxserver`, `inctarget`).
- `sshd` is not enabled by default post-install.
- A stale DHCP reservation from the scrapped first `zbx-server` build had
  to be cleaned from libvirt's `default` network after the rebuild.

## Zabbix on openSUSE Leap 16

- **No official Zabbix repo exists for Leap 16.0** — upstream RPMs depend
  on legacy `libcrypto.so.1.1`, which Leap 16's OpenSSL 3.x doesn't ship.
  Used openSUSE's own `server:monitoring` OBS project instead (Zabbix
  7.0.28, built specifically for 16.0).
- DB backend: MariaDB. Agent: Zabbix Agent 2 (both chosen deliberately).
- openSUSE packaging uses underscores where docs use hyphens:
  `zabbix_server.service`, `zabbix_agent2.service`; log path has a
  trailing "s": `/var/log/zabbixs/`.
- `zypper install zabbix-ui` pulled in `php8-pdo` but not an actual DB
  driver or curl — `php8-mysql`, `php8-curl`, `apache2-mod_php8` all
  needed manual installation afterward.
- Apache needs `APACHE_SERVER_FLAGS="ZABBIX"` in `/etc/sysconfig/apache2`
  to activate the frontend's `<IfDefine ZABBIX>` block.
- The shipped `zabbix.conf` only explicitly *denies* `conf/`/`include/`
  and never explicitly *grants* `/usr/share/zabbix` itself — needed an
  added `<Directory>` block with `Require all granted`.
- PHP's Apache module registers internally as `php_module`, not
  `mod_php8.c` — `<IfModule mod_php8.c>` guards silently never matched.
  The three required `php_value` overrides (`post_max_size`,
  `max_execution_time`, `max_input_time`) had to be set unconditionally.
- `/usr/share/zabbix/conf/zabbix.conf.php` had to be created manually
  (`root:root 644`) — Apache/PHP lacked write access to that directory,
  so the wizard's create *and* overwrite attempts both failed; the
  manually-written file was picked up fine regardless.
- SELinux (`Enforcing`) required two real fixes, confirmed via `ausearch`
  AVC entries: `httpd_can_network_connect` boolean turned on (outbound
  connections, including the frontend's own-server status check);
  `pcre.jit=0` added to `/etc/php8/apache2/php.ini` (SELinux blocks PHP's
  JIT from marking memory executable).
- `inc-target`'s `zabbix_agent2.conf` needed `Server=`/`ServerActive=`
  pointed at `zbx-server`'s IP, and `Hostname=` changed from the package
  default (`Zabbix server` — collides with the built-in self-monitoring
  host) to `inc-target`. An overly broad `sed` pattern briefly created
  duplicate `Server=`/`ServerActive=` lines — fixed via exact
  line-number targeting.
- Known, expected non-issue: `inc-target`'s virtio NIC reports no real
  link speed/type, so that one Zabbix item permanently shows "Not
  supported" — an artifact of virtualization, not a fault.

## fail2ban

- Default `banaction = iptables-multiport` doesn't work correctly here —
  firewalld, not raw iptables, manages this system's rules. Overridden to
  `banaction = firewallcmd-rich-rules` in `jail.local`.
- `journalctl -u fail2ban` only shows the systemd unit's own start/stop
  messages, not ban/unban activity — the real event log is
  `/var/log/fail2ban.log` (per `logtarget` in `fail2ban.conf`).
- openSUSE's fail2ban ships `paths-opensuse.conf`, setting
  `sshd_backend = systemd` (reads journald directly, no logfile needed).
- Confirmed end-to-end: full ban→auto-unban cycle verified with exact
  timing (banned/unbanned exactly 600s apart, matching configured
  `bantime`).
- sshd's `LoginGraceTime` can silently drop a slow/incomplete connection
  before any auth-failure line is logged, leaving fail2ban with nothing
  to match — not a fail2ban fault, but worth checking if a manual test
  doesn't trigger a ban.

## Snapper

- Full create→status→undochange cycle confirmed on `inc-target`: a test
  file created between two snapshots was correctly identified by
  `snapper status` and precisely reverted by `undochange`, verified via
  direct filesystem check afterward.

## Host resource constraints

- T490: 7.4GiB usable RAM. Both VMs at 2048MB baseline = 4GB already
  committed before any host-side overhead.
- Swap is zram (compressed, RAM-backed), not disk swap — pressure shows
  as increasing sluggishness (CPU spent compressing/decompressing), not
  a sudden crash, until the OOM killer becomes a last resort.
- Rule of thumb: watch the "available" column in `free -h` on the **host**
  (not "free") — persistently under ~1GB is the real warning line to
  check *before* deliberately spiking load (e.g. inc-105), not after.

## Process/workflow notes

- Several mistakes this project came from running a command on the wrong
  machine (host vs. one of the two VMs) — prompts can look superficially
  similar. `hostnamectl` (not `hostname`, which isn't installed on the
  minimal Leap image) is the reliable way to confirm which machine before
  running anything consequential.
- `<placeholder>`-style bracket syntax in example commands got typed
  literally more than once — literal, copy-pasteable commands are used
  instead now.
- Bundled multi-command pastes have repeatedly lost output partway
  through (a `sudo` password prompt interrupting, terminal truncation) —
  splitting into single commands is the reliable fix when a step is
  ambiguous or its result matters.
- Despite building all six incident runbooks and this gotchas doc across
  multiple sessions, none of it was ever `git add`ed or committed — the
  entire body of work sat as untracked files in the working tree until a
  deliberate file/git audit caught it. Lesson: commit each artifact right
  after creating it, not in a batch at the end.

## Spontaneous incident: brief NTP desync (inc-target, 2026-09-22)

- Real, unplanned incident — not staged. chronyd's 4 configured NTP
  sources briefly flipped offline→online within the same second
  (~16:52:18), plausibly correlating with the host resuming from a
  multi-day idle/suspend gap between work sessions.
- Zabbix correctly detected and alerted: "System time is out of sync
  (diff with Zabbix server > 60s)", Warning severity.
- chronyd self-corrected on its own polling cycle — confirmed via
  `chronyc tracking` showing sub-millisecond offset and
  `Leap status: Normal` shortly after. Zabbix problem auto-resolved with
  no manual intervention.
- Kept as evidence the monitoring stack correctly catches real, unstaged
  incidents too, not just deliberately engineered ones.

## zabbix_get error messages don't distinguish firewall-blocked from nothing-listening

  Both a firewalld reject and a service bound to loopback-only produce the
  identical `connection error (POLLERR,POLLHUP)` from `zabbix_get` — the
  client-side error text is not a reliable signal for which layer is
  actually broken. Verify target-side state directly instead (`ss -tlnp`
  for bind address, `firewall-cmd --list-all` / `nft list ruleset` for
  firewall rules) rather than inferring cause from how a connection attempt
  failed.

## Docker present on inc-target

  `firewall-cmd --get-active-zones` revealed a `docker` zone bound to
  `docker0` — Docker is installed and running here, unrelated to anything
  deliberately set up for this project. Not investigated further; flagged
  for awareness only.
