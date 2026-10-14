# `oldtown.engsoc.net` — Server Inventory and Operations Runbook

**Document status:** Initial inventory draft; validate before using for disaster recovery.  
**Inventory date:** 13 October 2026 (observations are from the command outputs collected during the inventory).  
**Operating system:** Debian GNU/Linux 12 (bookworm), x86-64.  
**Host:** `oldtown.engsoc.net` — HP ProLiant DL360 Gen9, firmware P89.

> **Important:** This document deliberately omits passwords, API keys, private keys, application secrets, and the contents of secret-bearing configuration files. Keep the completed runbook in access-controlled storage. This is an inventory, not a claim that every recovery path has been tested.

## 1. Executive summary

`oldtown` is a physical Debian server providing ZFS-backed storage, Samba file shares, NFS/RPC services, eleven libvirt virtual machines, and several Docker-hosted applications. Networking includes two active physical interfaces, a third configured/down interface, a libvirt bridge, Docker networks, and Tailscale. The host runs a Docker-aware Caddy reverse proxy for four web applications.

### Highest-priority operational risks

1. **Off-host ZFS replication is failing.** Root cron invokes `syncoid` every six hours, but `/var/backups/.ssh/backup` is missing. Logs report an inaccessible identity file and SSH authentication failure. Do not assume the remote replica is current.
2. **Samba domain trust/authentication is failing.** SSSD reports the FreeIPA domain online, but Samba/winbind domain-controller and trust checks fail. SMB access from a Windows client has also failed.
3. **UPS shutdown behavior is unverified.** A Tripp Lite SMART1500RM2U is connected over USB, but no UPS monitoring daemon/package was identified. Graceful shutdown on battery depletion is not established.
4. **Kernel update awaits reboot.** Kernel `6.1.0-53-amd64` is installed, but the running kernel is `6.1.0-48-amd64`. Schedule a maintenance window after reviewing workload and recovery readiness.
5. **OpenSCAP findings need validation.** The daily scan reports 13 Python 3.11 CVE findings. The installed package is `3.11.2-6+deb12u8`, which is the configured APT candidate in the captured output. Investigate scanner definitions, package applicability, and report details before concluding the findings remain unresolved.
6. **The OpenSCAP job applies remediation.** The daily script uses `oscap ... --remediate`, deletes prior reports at the start, and does not currently email reports because the email command is commented out. This is a change-making security job, not report-only monitoring.
7. **Audit events have been lost.** `auditctl -s` reported `lost 3`; the kernel journal also recorded an audit netlink buffer-space error.
8. **Host naming is inconsistent.** `/etc/hostname` says `oldtown.engsoc.net`, while `hostname -f` returned `localhost`; `/etc/hosts` includes old/stale aliases.
9. **Firewall state is assembled by multiple components.** UFW is enabled but inactive; nftables service is disabled, while Tailscale, Docker, libvirt and `/root/firewall.sh` contribute live rules. Review effective rules, not just one firewall frontend.
10. **Application recovery is not verified.** ZFS snapshots exist, but this is not proof that application databases, uploaded files, secrets, and deployment configuration can be restored together.

## 2. Hardware and operating system

- **Platform:** HP ProLiant DL360 Gen9, firmware P89.
- **CPU:** 2 × Intel Xeon E5-2680 v4 @ 2.40 GHz; 14 cores/socket, 2 threads/core; 56 logical CPUs; 2 NUMA nodes.
- **Memory:** approximately 314 GiB RAM; no swap configured.
- **OS:** Debian 12 (bookworm), x86-64.
- **Running kernel at inventory:** `6.1.0-48-amd64`.
- **Installed kernel packages:** `6.1.0-48-amd64` and `6.1.0-53-amd64`; needrestart identified `6.1.0-53-amd64` as expected kernel.
- **Uptime at inventory:** approximately 91 days since boot on 13 July 2026.
- **Time synchronization:** chrony active.
- **Hostname issue:** `/etc/hostname` contains `oldtown.engsoc.net`, but `hostname -f` returned `localhost`. `/etc/hosts` maps `127.0.0.1` to `localhost`, `oldtown`, `oldtown.engsoc.net`, and `openslides.mycses.ca`; `127.0.1.1` maps to `cses2`. Verify intended canonical hostname and aliases before changing them.

## 3. CPU, memory and disk capacity

- RAM: about 314 GiB total, about 149 GiB available in the captured output.
- No swap was configured.
- `bpool`: 7.50 GiB total, around 631 MiB allocated at inventory.
- `rpool`: 7.27 TiB total, around 1.42 TiB allocated at inventory.
- `rpool` allocation differs materially from the approximate dataset-used total recorded during inventory; investigate snapshots, metadata, reservations, and accounting if capacity planning requires an exact explanation.

## 4. ZFS and physical storage

### Pools

Both pools are `ONLINE`, use a single eight-disk RAIDZ2 vdev, and showed zero READ/WRITE/CKSUM errors and no known data errors. Both were scrubbed on 11 October 2026:

- `bpool`: scrub repaired 0 B in 5 seconds.
- `rpool`: scrub repaired 0 B in 1 hour 19 minutes 46 seconds.

Each of the eight disks contributes partition 2 to `bpool` and partition 3 to `rpool`. Stable WWN partition paths are used. `ashift=12`, `autotrim=on`, `autoreplace=off`, `autoexpand=off`. `bpool` reports that some supported pool features are not enabled. **Do not run `zpool upgrade` casually**; it may make the pool inaccessible to older ZFS implementations.

All eight SMART overall-health self-assessments (`/dev/sda` through `/dev/sdh`) returned `PASSED`. This is a useful baseline, not a guarantee that disks will not fail. Preserve and review SMART attributes and test history periodically.

### Important datasets

| Dataset | Mountpoint | Approximate used |
|---|---|---:|
| `rpool/ROOT/debian` | `/` | 74.1 GiB |
| `rpool/home` | `/home` | 25.9 GiB |
| `rpool/home/root` | `/root` | 16.4 GiB |
| `rpool/srv/bastion` | `/srv/bastion` | 4.08 GiB |
| `rpool/srv/share` | `/srv/share` | 874 GiB |
| `rpool/srv/share/Archive` | `/srv/share/Archive` | 706 GiB |
| `rpool/srv/share/CPRT` | `/srv/share/CPRT` | about 682 KiB |
| `rpool/var/lib/docker` | `/var/lib/docker` | 34.3 GiB |
| `rpool/var/log` | `/var/log` | 9.71 GiB |

`/srv/share/Archive` is the largest identified data set. Samba's `Archive` share uses `force user = root`, so file operations through that share are performed as root; review whether this is required.

### Snapshots and replication

Sanoid snapshots are configured for `bpool` and `rpool` using the `production` template: hourly 36, daily 30, monthly 3, yearly 0; autosnapshot and autoprune enabled. A separate `rpool/cses3` configuration uses the `backup` template (hourly 30, daily 90, monthly 12) with autosnapshot disabled.

Root cron attempts recursive syncoid replication every six hours:

- `rpool` → `backup@10.6.0.5:rpool/cses2/rpool`
- `bpool` → `backup@10.6.0.5:rpool/cses2/bpool`
- SSH identity configured as `/var/backups/.ssh/backup`
- Logs: `/var/log/sync_rpool.log`, `/var/log/sync_bpool.log`

**Observed failure:** `/var/backups/.ssh` was empty. Latest captured logs reported the identity file missing, too many authentication failures, and SSH test failure (exit 255). TCP/22 on `10.6.0.5` was reachable. Repair the replication identity/access with the remote administrator, then verify a successful run and compare destination snapshots. Do not create or copy keys without confirming the intended key and remote authorization.

Local ZFS snapshots are not a substitute for an off-host backup. No verified application-level backup/restore procedure was identified. Run restore tests before treating any backup as reliable.

## 5. Network topology

### Interfaces and routes

- `eno1`: UP, `10.7.0.6/24`; default route via `10.7.0.5`.
- `eno49`: UP, `10.6.0.6/24`.
- `eno50`: DOWN, configured `10.5.0.6/24`.
- `virbr10`: UP, `10.9.0.1/24`; used by libvirt guests.
- Tailscale `tailscale0`: `100.64.0.2/32` plus Tailscale IPv6 address.
- Multiple Docker bridge networks (including `caddy-net` at `10.50.0.0/24`) and virtual interfaces.
- Tailscale peers include `driftmark.engsoc.net` (`100.64.0.6`) and `dragonstone.engsoc.net` (`100.64.0.7`).
- DNS uses systemd-resolved local stub (`127.0.0.53`) and Quad9 upstream resolvers; Tailscale DNS is also configured.

### Host listeners of operational interest

Captured listeners included:

| Port/protocol | Bind | Service / purpose |
|---|---|---|
| TCP 22 | all IPv4/IPv6 | SSH |
| TCP 80 | all IPv4/IPv6 | Docker-published Caddy HTTP ingress |
| TCP 8000 | all IPv4 | OpenSlides proxy |
| TCP 445, 139 | all IPv4/IPv6 | Samba SMB/NetBIOS session service |
| UDP 137, 138 | multiple interfaces | Samba NetBIOS name/datagram service |
| TCP 111, UDP 111 | all IPv4/IPv6 | rpcbind |
| TCP 2049 | all IPv4/IPv6 | NFS |
| TCP/UDP dynamic RPC ports | all or IPv6 | mountd/statd and related RPC services |
| TCP 9090 | wildcard socket | Cockpit, socket-activated |
| TCP 5002 | all IPv4 | Houston alert webhook |
| TCP 5355 / UDP 5355 | all IPv4/IPv6 | LLMNR via systemd-resolved |
| TCP 25 | loopback only | Postfix SMTP |
| TCP 53 | loopback | systemd-resolved stub |
| TCP 5900, 5901 | loopback only | QEMU VNC |
| TCP 9002 | loopback only | OpenSlides backendManage published locally |
| TCP 42780 | loopback only | Redis listener observed locally |

This list is a snapshot, not a complete firewall exposure assessment. In particular, no host TCP/443 listener was observed; Caddy listens on host port 80 and has automatic HTTPS disabled. Determine which upstream ingress device/service provides TLS and whether external network ACLs protect these ports.

### Firewall

- UFW service is enabled but UFW itself reports inactive.
- `nftables.service` is disabled/inactive, but live rules exist from Tailscale, Docker, libvirt and the custom `/root/firewall.sh`.
- Root crontab runs `/root/firewall.sh` at boot.
- The script enables IPv4 forwarding, flushes the iptables filter table, adds source/destination and port rules, sets `FORWARD` policy to DROP, applies virbr10 restrictions, and configures NAT/MASQUERADE rules. It does not set `INPUT` policy to DROP.
- `/root/firewall-testing.sh` is an unexecuted experiment that flushes filter and NAT tables and changes routing. **Do not execute it during routine operation.**
- IPv4 forwarding is enabled in sysctl configuration.

Because multiple services manage firewall rules, record `nft list ruleset`, `iptables-save`, `ip6tables-save`, Docker port mappings, and upstream ACLs together when refreshing this section. Avoid changing firewall rules remotely without console/recovery access.

## 6. Docker and web applications

### Reverse proxy

Container `caddy` uses `lucaslorentz/caddy-docker-proxy:alpine`, publishes host port 80, and attaches to `caddy-net` and the OpenSlides frontend network. It mounts the Docker socket read-write, which effectively grants the proxy container powerful control over the Docker host. Treat Caddy configuration and container access as highly privileged.

Caddy automatic HTTPS is disabled in the captured configuration. Routes observed:

| Hostname | Upstream |
|---|---|
| `bitwarden.engsoc.net` | Vaultwarden at `10.50.0.4:1500` |
| `openslides.mycses.ca` | `openslides-proxy-1:8000` |
| `docs.engsoc.net` | BookStack at `10.50.0.2:80` |
| `sign.mycses.ca` | Documenso at `10.50.0.3:3000` |

Trusted proxy settings include `10.8.0.1` and a custom client IP header `X-Engsoc-Ingress-Ip`. Confirm that only trusted ingress infrastructure can set that header and that clients cannot spoof it.

### Running containers

| Service | Image / notes | Persistence / network notes |
|---|---|---|
| BookStack (`bookstack-int`) | `lscr.io/linuxserver/bookstack:latest` | `/root/compose/bookstack-internal/data` → `/config`; `.envfile` mounted into app (secret; do not publish); caddy-net and `bookstack-int` |
| MariaDB for BookStack | MariaDB 11.4.8 | `/root/compose/bookstack-internal/db` → `/config`; internal `bookstack-int` network |
| Documenso | `documenso/documenso:latest` | `/root/compose/documenso/certificate.p12` mounted read-only; caddy-net plus private Documenso network |
| Documenso PostgreSQL | PostgreSQL 15 | `/root/compose/documenso/db` → PostgreSQL data directory |
| Vaultwarden | `vaultwarden/server:latest` | `/root/compose/vaultwarden/data` → `/data`; caddy-net |
| OpenSlides | version 4.3.0, 14-container stack | PostgreSQL data volume `openslides_postgres-data`; secrets under `/root/compose/openslides/secrets/`; several internal networks; proxy published on TCP 8000 |
| Caddy | `lucaslorentz/caddy-docker-proxy:alpine` | Docker socket read-write; Caddy data/config under `/root/compose/caddy/` |

Additional OpenSlides containers observed: proxy, client, projector, backendAction, backendPresenter, vote, search, media, auth, backendManage, icc, autoupdate, PostgreSQL 17.10 and Redis Alpine. Most were healthy at the time of inventory.

Compose projects observed under `/root/compose`: `bookstack-internal`, `caddy`, `documenso`, `egov`, `minecraft`, `openslides`, and `vaultwarden`; an older reverse-proxy project is under `/root/docker/reverse-proxy`. Several old containers are stopped. Do not delete stopped containers or compose files until their purpose and data ownership have been confirmed.

## 7. Virtual machines and libvirt

`libvirtd` and `virtlogd` are active. Eleven persistent guests were running at inventory; all were reported AppArmor-enforcing and attached to `virbr10`.

| VM | vCPU | Memory reported | Disk image | Autostart | Guest address observed |
|---|---:|---:|---|---|---|
| `froshcraft` | 8 | 20 GiB used of 32 GiB max | `froshcraft.qcow2` | yes | not found in captured lease/ARP results |
| `chatbot` | 4 | 4 GiB | `chatbot.qcow2` | yes | `10.9.0.20` |
| `ravens-racing` | 2 | 4 GiB | `ravensracing.qcow2` | yes | `10.9.0.10` |
| `scesoc-vm` | 4 | up to 8 GiB | `scesoc.qcow2` | yes | `10.9.0.8` |
| `docker-dev` | 1 | 2 GiB | `docker-dev.qcow2` | yes | `10.9.0.153` |
| `bitwarden-dev` | 2 | 4 GiB | `bitwarden-dev-1.qcow2` | yes | `10.9.0.108` |
| `ipa-test` | 1 | 2 GiB | `ipa-test.qcow2` | yes | `10.9.0.136` |
| `bastion` | 2 | 2 GiB | `bastion.qcow2` | yes | `10.9.0.244` |
| `archive-dev` | 1 | 2 GiB | `archive-dev.qcow2` | yes | `10.9.0.29` |
| `squashclub` | 2 | 2 GiB | `squashclub.qcow2` | yes | `10.9.0.30` |
| `myvm` | 2 | 4 GiB | CD-ROM ISO at `/home/bekkers/debian-13.6.0-amd64-netinst.iso` | no | not found in captured lease/ARP results |

Guest IPs are observations, not necessarily static reservations. Confirm addressing inside each guest or through authoritative DHCP/DNS. `virbr10` is configured as a bridge network (not a typical libvirt NAT network), with host address `10.9.0.1/24`.

Libvirt storage pools observed:

- `almalinux-9`: inactive, autostart enabled, path `/root/images/almalinux-9`.
- `almalinux-9-1`: active, autostart enabled, path `/home/mircea/images/almalinux-9`.
- `bekkers`: active, autostart enabled, path `/home/bekkers`.
- `default`: active, autostart enabled, path `/var/lib/libvirt/images`.

Some QCOW2 images and XML definitions did not clearly correspond to running guests (including `directory-dev.qcow2`, `CSESDefault.qcow2`, and several XML files). Treat these as unclassified assets, not safe-to-delete leftovers.

## 8. Identity, SSH, Samba and file services

### FreeIPA / SSSD

- SSSD domain: `engsoc.net`; `sssctl config-check` passed.
- SSSD reported domain online with active server `dragonstone.engsoc.net`; `driftmark.engsoc.net` was also discovered.
- Example identity `bekkers` resolves via NSS with UID 5026, GID 20011 (`internal`), home `/home/bekkers`, shell `/bin/bash`.
- Related services include SSSD, winbind, nslcd, oddjobd, Samba smbd/nmbd and an enabled `samba-ad-dc` unit. The latter is enabled although Samba reports this host is a member server; verify whether that unit is intentionally enabled.

### Samba

Samba reports workgroup `ENGSOC`, realm `ENGSOC.NET`, member-server role, and `map to guest = Never`. Shares:

- `[homes]`: writable home shares.
- `[BastionShare]`: `/srv/bastion`, writable, valid user `bastion`.
- `[CPRT]`: `/srv/share/CPRT`, writable, valid user `cprt`.
- `[Archive]`: `/srv/share/Archive`, writable, valid users include `@archive`, `mircea`, `bekkers`; `force user = root`.
- `IPC$`: standard IPC share.

Anonymous listing of share names on localhost succeeded. SMB1 is disabled. Samba reported a weak-crypto warning from GnuTLS regarding NTLM fallback.

**Domain issue:** `net ads testjoin`, `wbinfo --ping-dc`, and `wbinfo -t` failed with domain-controller / logon-server errors, even though SSSD is online. A Windows client connected to TCP 445/139 over Tailscale but `net view` failed with `NT_STATUS_NO_LOGON_SERVERS`. Investigate Samba/winbind trust, DNS, time, machine account and domain-controller reachability without rejoining or changing trust until credentials and recovery plan are confirmed.

### SSH

SSH is enabled and listening on all IPv4/IPv6 interfaces. Effective settings observed:

- `PermitRootLogin without-password` (public-key only for root under this setting).
- Public-key authentication enabled.
- Password authentication enabled.
- Keyboard-interactive authentication enabled.
- X11 forwarding and TCP forwarding enabled.
- No `AllowUsers`, `AllowGroups`, or deny-list restrictions were found in the inspected effective output.

Review exposure and authentication policy against operational needs. Do not lock down authentication remotely without confirming alternate access and testing a second session.

### NFS and RPC

NFS and RPC services are active/listening (including NFS TCP 2049 and rpcbind/mountd/statd dynamic ports), but `/etc/exports` contained only examples, `/etc/exports.d` was absent, and `exportfs -v` was empty. No NFS/CIFS/SSHFS remote mounts were found. Confirm whether the NFS server is intended to export anything; otherwise document why the service remains enabled before disabling it.

`rpc-svcgssd.service` is the only failed systemd unit in the captured status. `/etc/default/nfs-kernel-server` had `NEED_SVCGSSD=""`; investigate whether the failure is harmless for the configured NFS mode.

## 9. Security, monitoring and maintenance

### OpenSCAP

- Script: `/root/openscap/scan.sh`.
- Schedule: root cron daily at 01:01.
- Downloads Debian Bookworm OVAL definitions, removes prior XML and HTML reports, runs `oscap oval eval --remediate`, and writes `report.html`; if findings are detected, runs a second evaluation to `report_fixed.html`.
- The email notification command is commented out.
- Captured reports contained 13 Python 3.11 vulnerability findings and 57,776 compliant results, with zero errors in the summary.
- Installed Python packages included `python3.11` and `python3.11-minimal` version `3.11.2-6+deb12u8`; APT candidate was also `...deb12u8` in captured policy output.

**Operational concern:** `--remediate` can change system state. Review the script's intended behavior, permissions, OVAL findings and remediation logs. Preserve reports if audit history is required; the current script deletes previous reports each run. Do not rerun the remediation scan as a diagnostic without approval.

### Audit and malware monitoring

- `auditd` enabled and active. Audit status reported `lost 3`, backlog limit 8192, backlog 0. Journal recorded `Error receiving audit netlink packet (No buffer space available)` on 13 October 2026. Investigate audit rate/buffer sizing and confirm log retention.
- Linux Malware Detect (maldet) v1.6.6 service active with inotify monitoring.
- `smartmontools` enabled/active; configuration scans all devices and can email `technical@cses.carleton.ca`.
- Houston ZFS capacity alert timer active hourly; threshold 80%; last observed checks reported bpool 8% and rpool 19% used. Logs at `/tmp/zfs_storage_alert.log`, state at `/var/lib/45drives/zfs_alert_state.json`.
- Houston webhook service runs `/usr/bin/python3 /etc/45drives/alerts-webhook.py` as root and listens on TCP 5002 on all interfaces. Treat the listener as an exposed service; verify firewall and ingress expectations.

### APT updates and reboot

`apt-daily.timer` and `apt-daily-upgrade.timer` are enabled and waiting. The captured query did not show effective APT periodic configuration values or an installed `unattended-upgrades` package, so automatic security installation is **not confirmed** merely because the timers are active. Recent package history shows a large `apt upgrade -y` on 5 October 2026.

Kernel `6.1.0-53-amd64` was installed, while `6.1.0-48-amd64` was running; `needrestart` reported kernel status `3` (reboot required). No `/var/run/reboot-required` marker existed, but that does not negate the needrestart kernel result. Schedule and perform a controlled reboot after checking VM/application startup, backup status, console access and service owners.

### UPS

A Tripp Lite SMART1500RM2U UPS is attached over USB (`09ae:3015`). The kernel repeatedly recognized it. No NUT, apcupsd, PowerPanel or comparable UPS monitoring package/service was identified. No low-battery shutdown path has been verified. Establish whether the UPS has network management or USB monitoring intended, then test alerts and controlled shutdown in a maintenance window.

## 10. Key paths and schedules

| Path / unit | Purpose |
|---|---|
| `/root/compose/` | Main Docker Compose configurations and associated application data/configuration |
| `/root/compose/caddy/` | Caddy Docker proxy configuration and data |
| `/root/compose/openslides/` | OpenSlides compose files and secret directory (do not publish secrets) |
| `/root/compose/bookstack-internal/` | BookStack and MariaDB data/configuration |
| `/root/compose/documenso/` | Documenso configuration and PostgreSQL data |
| `/root/compose/vaultwarden/` | Vaultwarden data |
| `/root/firewall.sh` | Boot-time firewall/routing script |
| `/root/firewall-testing.sh` | Experimental destructive firewall script; do not run casually |
| `/root/openscap/scan.sh` | Daily OVAL scan with remediation |
| `/etc/45drives/zfs-storage-check.sh` | ZFS capacity threshold checks |
| `/etc/45drives/alerts-webhook.py` | Houston alert webhook |
| `/var/log/sync_rpool.log` | `rpool` replication log |
| `/var/log/sync_bpool.log` | `bpool` replication log |
| `houston-zfs-storage-alert.timer` | Hourly ZFS capacity check |
| `sanoid-prune.service` / Sanoid timer | Snapshot retention/pruning |

Root cron jobs captured:

- At reboot: `/root/firewall.sh`.
- Every six hours: recursive `syncoid` replication for `rpool` and `bpool` to `backup@10.6.0.5` (currently failing due to missing identity file).
- Daily midnight: `/usr/sbin/sanoid --cron`.
- Daily 01:01: `/root/openscap/scan.sh`.
- Monthly: delete files under `/var/backups` older than 120 days.

## 11. Failed or potentially stale services/configuration

- `rpc-svcgssd.service` is failed.
- `openvpn.service` is enabled but shown as `active (exited)`; no active tunnel was proven.
- UFW is enabled but inactive; nftables service is disabled while other tools install live rules.
- `samba-ad-dc.service` is enabled while Samba's role is member server.
- `wpa_supplicant` and `ModemManager` are active on a server; confirm whether needed before considering changes.
- NFS/RPC daemons listen despite no configured exports.
- Multiple stopped Docker containers and older Compose projects exist; classify before cleanup.
- Some VM images/XML definitions are not clearly tied to running guests.

## 12. Recommended action register

| Priority | Action | Why / acceptance criterion |
|---|---|---|
| Critical | Restore off-host ZFS replication | Missing SSH identity fixed securely; both pools replicate successfully; remote snapshot freshness is verified and documented. |
| Critical | Establish application backup and restore tests | Databases, uploads, configuration and secrets can be restored together to a test location. |
| High | Diagnose Samba domain trust | `wbinfo`/`net ads testjoin` and a real client share access succeed, or the service is formally retired. |
| High | Verify UPS monitoring/shutdown | UPS status is monitored and low-battery behavior is documented and safely tested. |
| High | Plan reboot into kernel 6.1.0-53 | Maintenance window, console access and post-reboot service checks completed. |
| High | Review OpenSCAP remediation behavior and findings | Findings validated against package/security metadata; report retention and alerting policy defined. |
| High | Review effective firewall and external ingress | Approved network diagram, TLS termination, source ACLs, exposed ports and trusted proxy headers documented. |
| Medium | Resolve hostname inconsistency | `hostname -f`, `/etc/hostname`, `/etc/hosts`, DNS and certificates agree on intended names. |
| Medium | Investigate audit loss | Root cause of buffer overflow understood; audit event loss no longer observed under normal load. |
| Medium | Review SSH authentication and Cockpit exposure | Access paths and allowed source networks documented and tested. |
| Medium | Decide whether NFS, OpenVPN, samba-ad-dc, ModemManager and wpa_supplicant are required | Each service has a documented owner/purpose; no removals until verified. |
| Medium | Explain ZFS allocation accounting difference | Pool allocation, snapshots, metadata and dataset totals reconciled for capacity planning. |
| Low | Classify stopped containers and unused-looking VM assets | Each is labeled active, retained for rollback, archive, or safe-to-remove after owner approval. |

## 13. Refresh commands for future inventory

Run read-only checks as root or with `sudo`; avoid dumping secrets or entire environment variables. Capture output with date and host identity.

```bash
 date -Is
 hostnamectl
 hostname -f
 cat /etc/os-release
 uname -a
 lscpu
 free -h
 lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
 zpool status -xv
 zpool list
 zfs list -o name,used,avail,refer,mountpoint
 zpool get all bpool rpool
 smartctl --scan-open
 systemctl --failed --no-pager
 systemctl list-units --type=service --state=running --no-pager
 systemctl list-timers --all --no-pager
 ip -br address
 ip route
 ss -lntup
 sudo nft list ruleset
 sudo iptables-save
 sudo ip6tables-save
 sudo docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
 sudo docker compose ls
 sudo virsh list --all
 sudo exportfs -v
 sudo sssctl domain-status engsoc.net
 sudo auditctl -s
 sudo needrestart -r l
```

For each Docker application and VM, maintain: service owner, purpose, users/URLs, source of truth for configuration, persistent data paths, secrets location (path only), dependencies, ports, backup method, restore procedure, monitoring, upgrade method, and rollback plan.

## 14. Completion checklist for a true operational runbook

This first draft is broad, but “100% documented” requires owner validation and a few missing sources of truth:

- [ ] Confirm intended role and owner for every public-facing service and VM.
- [ ] Draw physical/network/VLAN/Tailscale/ingress diagram and record upstream TLS termination and ACLs.
- [ ] Export effective service configuration without secret values; version-control sanitized copies.
- [ ] Document DNS records, registrar/authoritative DNS ownership, and ingress dependencies.
- [ ] Repair and verify off-host ZFS replication; document remote host ownership and recovery commands.
- [ ] Document and test application-consistent backups and full restore for each critical service.
- [ ] Record UPS monitoring and shutdown policy.
- [ ] Reconcile the firewall ruleset and the purpose of all listening ports.
- [ ] Document VM guest OS versions, guest administrators, static IPs/DHCP/DNS, applications and backup/restore.
- [ ] Document certificate/TLS renewal and external proxy responsibilities.
- [ ] Record service owners/contact/escalation, maintenance windows, alert destinations and incident response.
- [ ] Perform a controlled reboot and post-reboot smoke test.
- [ ] Date, review, and store this runbook somewhere available if `oldtown` is unavailable.
