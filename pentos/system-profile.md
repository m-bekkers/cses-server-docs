# System Profile

**Captured:** 2026-10-13 local time  
**Host:** `pentos.engsoc.net`  
**Confidence:** Direct host observations unless explicitly marked as repository-derived.

## Identity and platform

| Property | Observed value |
| --- | --- |
| Chassis | HP ProLiant DL20 Gen9 server |
| Firmware | U22 |
| Operating system | Debian GNU/Linux 12 (Bookworm) |
| Kernel | `6.1.0-39-amd64` (reported kernel build `6.1.148-1`) |
| Architecture | x86-64 |
| Time zone | `America/Toronto` (EDT, UTC-04:00 at capture) |
| Clock | Synchronized; chrony active |
| Host account used for inventory | `bekkers` (`/home/bekkers`) |

Do not use the machine ID or boot ID as identifiers in external tickets or documentation.

## Compute and memory

| Resource | Observed value |
| --- | --- |
| CPU | Intel Xeon E3-1240 v5 @ 3.50 GHz |
| Topology | 1 socket, 4 cores, 2 threads/core; 8 logical CPUs |
| Memory | 62 GiB total; about 32 GiB available at capture |
| Swap | 975 MiB total; about 972 MiB free at capture |
| Virtualization | VT-x capability present; host is reported as a physical server |

## Storage and filesystems

Two WDC WD1005FBYZ 1 TB-class disks provide a Linux `md` RAID1 array. `/dev/md0` comprises `/dev/sda2` and `/dev/sdb2`; `/proc/mdstat` reported `[2/2] [UU]` at capture. The array is formatted as ext4 and mounted as `/`.

| Mount / device | Filesystem | Capacity | Used | Available | Notes |
| --- | --- | ---: | ---: | ---: | --- |
| `/` (`/dev/md0`) | ext4 | 911 GiB | 825 GiB | 40 GiB | 96% used; capacity warning |
| `/boot/efi` (`/dev/sdb1`) | vfat | 511 MiB | 6 MiB | 506 MiB | EFI system partition |
| swap (`/dev/sda5`) | swap | 976 MiB | 4 MiB | 972 MiB | Small swap area |

The `lsblk` view also showed an unmounted 4.7 GiB vfat partition on `/dev/sda3` and an unmounted 976 MiB partition on `/dev/sdb3`; their purpose was not established. Check the partition table and boot configuration before changing them.

### Capacity response

Treat the 96% root utilization as a near-term maintenance item. Identify the largest paths first (`du -x`, `ncdu` if installed, and Docker disk-usage inspection with authorized access). Preserve database files, Docker volumes, backups, and logs until owners and recovery copies are confirmed. RAID1 protects against one disk failure, not accidental deletion, corruption, ransomware, or host loss.

## Network

The primary active physical interface observed was `eno2`, with `10.7.0.7/24` and default gateway `10.7.0.5`. `eno1` was down and had a link-local address. Tailscale was active with `100.64.0.3/32`; treat this as a dynamic overlay address, not a stable identity. The host was using Docker bridge networks, including observed subnets `10.5.0.0/24`, `10.10.10.0/24`, and several `172.17.0.0/16` through `172.24.0.0/16` networks. Docker network names and bridge IDs were not mapped.

Local DNS stub listeners were present on `127.0.0.53` and `127.0.0.54`. Firewall rules, upstream routing/ACLs, public DNS, and external reachability were not inspected.

## Software baseline

| Component | Observed version / state |
| --- | --- |
| Debian package records | 1,325 installed package records at capture |
| Docker Engine | 29.8.2 |
| Docker Compose | 5.6.0 |
| Python | 3.11.2 |
| Node.js | 18.20.4 |
| Git | 2.39.5 |
| MariaDB | 10.11.18 service reported running |
| PostgreSQL | `18-main` cluster service reported running |
| Redis | `redis-server` service reported running |

An exact package manifest was not captured. Generate one with `dpkg-query -W -f='${binary:Package}\t${Version}\n'` during a future controlled refresh and review it for internal package names before sharing.

## Access and system management

The observed login account belonged to groups `internal`, `pubsnas`, `archive`, and `admins`. `/var/run/docker.sock` was `root:docker` with mode `0660`; the account was not a member of `docker`, so direct Docker API access failed. Non-interactive `sudo` also required a password. Do not add users to the Docker group casually: membership effectively grants root-equivalent host control.

Authentication was integrated with SSSD; the main SSSD daemon was running while six responder/socket units were failed at capture. See [Operations and recovery](operations.md).