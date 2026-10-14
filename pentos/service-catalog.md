# Service Catalog

This catalog is a point-in-time snapshot from systemd and socket inspection on 2026-10-13. A listener is not proof that a port is reachable from the public internet; host firewall and network ACLs were not inspected. Some process names were unavailable to the unprivileged inventory account.

## System services observed running

| Service | Observed role / notes |
| --- | --- |
| `docker`, `containerd` | Container runtime; active container inventory unavailable to this account |
| `ssh` | OpenSSH server; TCP 22 and 2222 were listening on wildcard addresses, but ownership of 2222 was not confirmed |
| `mariadb` | MariaDB 10.11.18; service active |
| `postgresql@18-main` | PostgreSQL 18 cluster; service active |
| `redis-server` | Redis service active; observed TCP 6379 on loopback and an additional loopback listener |
| `tailscaled` | Tailscale node agent; interface/address observed |
| `chrony` | Time synchronization active; system clock synchronized |
| `sssd` | Identity service active; responder failures listed below |
| `postfix@-.service` | Postfix service active; SMTP listener observed on loopback only |
| `systemd-resolved` | Local DNS stub listeners observed |
| `cron` | Scheduled-job service active |
| `smartmontools`, `mdmonitor` | Disk health and software RAID monitoring services active |
| `NetworkManager`, `networking`, `wpa_supplicant` | Network management components active |
| `cups`, `cups-browsed`, `avahi-daemon` | Printing and local discovery services active |
| `certmonger` | Certificate monitoring service active |
| `ModemManager`, `oddjobd`, `polkit`, `udisks2` | Additional system services active |

Other active units included D-Bus, journald, logind, udev, hostnamed, getty, and the user manager. Enabled units also included `openvpn`, `sysstat`, `apparmor`, `smartmontools`, and other base services; enabled does not mean currently active.

## Listening socket snapshot

| Bind address | Port / protocol | Observation |
| --- | --- | --- |
| `0.0.0.0`, `::` | TCP 22 | SSH listener; confirm intended access policy |
| `0.0.0.0`, `::` | TCP 80 | HTTP listener; likely proxy/container path, owner not confirmed |
| `0.0.0.0`, `::` | TCP 2222 | Listener owner and purpose not confirmed |
| `0.0.0.0`, `::` | TCP 5432 | PostgreSQL service active; listener binds broadly, verify intended exposure and firewall policy urgently |
| `0.0.0.0` | TCP 9201 | Owner and purpose not confirmed |
| `*` | TCP 9090 | Owner and purpose not confirmed |
| `10.5.0.1`, `10.10.10.1` | TCP 3306 | MariaDB listeners on Docker bridge addresses |
| `127.0.0.1` | TCP 6379 | Redis listener on loopback |
| `127.0.0.1` | TCP 25, 631 | SMTP and CUPS listeners on loopback |
| `127.0.0.1` | TCP 8001, 8080 | Local-only listeners; owner not confirmed |
| `127.0.0.53`, `127.0.0.54` | TCP/UDP 53 | Local DNS resolver |
| wildcard | TCP/UDP 5355 | Link-local name resolution listener(s); verify need and scope |
| wildcard UDP | 68 | DHCP client listener(s) |
| wildcard UDP | 5353 | mDNS listener |
| `127.0.0.1` | UDP 323 | Chrony/NTP local control listener |
| `100.64.0.3`, Tailscale IPv6 | TCP listeners | Overlay-interface listeners; ports vary and owners were not fully attributable |

Several additional high UDP ports and loopback VS Code server ports were transient/process-specific. Re-run `ss -tulpn` with approved privileges for a current owner-to-port map. Do not publish an unreviewed raw socket dump.

## Failed units

At capture time, these six units were failed:

- `sssd-pac.service`
- `sssd-nss.socket`
- `sssd-pam-priv.socket`
- `sssd-pam.socket`
- `sssd-ssh.socket`
- `sssd-sudo.socket`

Investigate with `systemctl status` and `journalctl -u <unit>` using an authorized account. Confirm whether these responders are required by the configured identity provider before changing or disabling them.

## Applications documented in the working tree

Repository configuration under `/home/bekkers/working/` describes the following deployment assets. Their presence in a checkout does not establish that a particular container, version, configuration, or checkout is currently deployed. See [Application Architecture](application-architecture.md) for the component flows and known configuration discrepancy.

| Application | Documented architecture / paths |
| --- | --- |
| CSES Affiliates Portal (`tasks.mycses.ca`) | Static frontend plus FastAPI backend, SQLite user store, Google OAuth, and OpenProject API integration. Backend Compose maps container port 8000 to host `127.0.0.1:8001` and attaches to external `caddy-net`. The README documents frontend destination `/var/www/tasks.mycses.ca` and deploy script `/usr/local/sbin/deploy-tasks-site`; retained Nginx override mounts `/var/www/tasks`. **Destination mismatch is unresolved.** |
| OpenProject | Separate legacy Compose stack with web, worker, cron, seeder, PostgreSQL, memcached, proxy, autoheal, and Hocuspocus collaboration components. The root Compose file publishes the proxy on `${PORT:-8080}`; an override describes Caddy routing. These manifests are distinct from the portal API Compose file. |

Portal Compose configuration identifies host-managed config/data paths `/opt/tasks-portal/secrets/backend.env` and `/opt/tasks-portal/data/`; only paths are recorded here. Do not inspect or copy environment-file contents into this handbook. The running host's active mounts and secret source were not verified. The portal API and OpenProject service are separate components. The observed host-level MariaDB and PostgreSQL units are not evidence of the database engines or data stores used by any active container.

## Container access gap

The Docker daemon was active, but `docker ps` failed because the current account could not access `/var/run/docker.sock`; `sudo -n` required a password. Container names, image tags, health, published ports, volumes, and Compose project names therefore remain unknown. Have an authorized operator capture those details after reviewing output for sensitive labels and paths.
