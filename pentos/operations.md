# Operations and Recovery

This is a conservative operator checklist for the observed host. Use the site's approved change process and an authorized administrator account before making changes. Commands below are examples; paths and service state can change.

## Routine checks

Run these read-only checks and record the timestamp and any significant changes:

```sh
date --iso-8601=seconds
hostnamectl
uname -a
uptime
free -h
df -hT
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS,MODEL
cat /proc/mdstat
systemctl --failed --no-pager
systemctl --type=service --state=running --no-pager
ss -tulpn
```

For authorized privileged inspection, use the organization's approved elevation workflow:

```sh
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker compose ls
docker system df
nft list ruleset
```

Do not paste raw environment variables, `docker inspect` output, secret files, private keys, database connection strings, or unredacted command output into this handbook or a ticket.

## Current follow-up items

1. **Disk capacity:** Root was 96% full with about 40 GiB available. Find the largest filesystems and growth sources; determine log retention, database growth, and backup placement. Avoid `docker system prune`, deleting volumes, or removing backups until ownership and restore requirements are understood.
2. **Database exposure:** TCP 5432 listened on wildcard IPv4 and IPv6 addresses. Confirm PostgreSQL's intended clients, `listen_addresses`, authentication rules, host firewall, and upstream ACLs. Restrict exposure to required networks; never assume a wildcard bind is protected by the network perimeter.
3. **Other wildcard listeners:** Validate the purpose and policy for 22, 80, 2222, 9201, 9090, and 5355. Attribute each listener to a process/container and confirm only intended services are exposed.
4. **SSSD:** Six responder/socket units were failed while `sssd.service` was running. Check unit status and logs, then validate login, NSS, PAM, SSH key, and sudo integration requirements with the identity-service owner.
5. **OpenProject placeholders:** The checked-in Compose manifest has insecure fallback values for database/application/collaboration secrets. Verify actual deployment configuration overrides every placeholder and that no fallback was used in production. Rotate any real secret that may have matched a committed fallback. Do not record values here.
6. **Container inventory:** An authorized account must capture active Compose projects, image versions, health, published ports, named volumes, and restart policies. Confirm persistence and backup ownership before upgrades or cleanup.
7. **Backups and recovery:** Backup schedules, destinations, retention, encryption, and restore-test evidence were not verified. Document these with the service owner. The repository's OpenProject guide describes backup commands, but that is not proof a backup job is running or current.

## Application maintenance

The portal repository documentation says the static frontend deployment and backend deployment are separate. The documented frontend deploy script is `/usr/local/sbin/deploy-tasks-site`; inspect its current contents and confirm the source checkout/branch before executing it. Backend updates use the repository's `docker-compose.backend.yml`; preserve host-managed config and data and take verified backups before upgrades.

For OpenProject, use the deployment's actual Compose files and supported upgrade procedure. The checked-in legacy documentation describes taking a PostgreSQL/assets backup before upgrades. Confirm current Compose project and volume names with an authorized Docker operator; do not guess or remove volumes.

### Change preparation and completion

Before a portal or OpenProject change, capture the active checkout and revision, Compose project name, exact Compose file list, image tags/digests, mounted paths/volumes, and current health. The checked-in documentation and manifests are not sufficient to identify these live values. Record the change owner, expected impact, backup location, and rollback decision point in the approved change record. Confirm a recent backup can be read and that the rollback procedure matches the current database version before a database or application upgrade.

After the change, verify the expected containers are healthy, review relevant logs for startup errors, check the public portal and request/authentication flow, and confirm the OpenProject integration if it was affected. A successful `/health` response checks only the API process according to the current code; it does not verify Google OAuth, SQLite writes, or OpenProject connectivity. Record the result and any follow-up in the change record without including secret values.

No verified rollback steps, backup retention policy, restore-test results, service owner, or recovery objectives were present in the reviewed material. Treat these as service-owner follow-ups; do not infer them from the repository's backup scripts.

## Incident response notes

- **Disk full:** Identify filesystem and top-level consumers with `df -hT` and `du -xhd1 /` (run with approved privileges where needed). Check logs and databases before acting. Preserve service data and backups.
- **RAID degradation:** Check `/proc/mdstat`, `mdadm --detail /dev/md0`, and SMART health. Do not remove or re-add devices based on a single observation; follow the storage owner's replacement procedure. RAID is not a backup.
- **Service failure:** Check `systemctl status <unit>` and `journalctl -u <unit> --since today`. Preserve relevant logs before restart if diagnosing a recurring fault.
- **Network exposure:** Correlate `ss -tulpn`, container-published ports, host firewall rules, and upstream ACLs. Binding to a wildcard address alone does not establish external reachability.
- **Security event:** Follow organizational incident response, isolate only with the responsible operator, preserve logs/evidence, and rotate credentials through their source-of-truth system. Do not place credentials in this documentation tree.

## Refreshing the package inventory

To capture the installed Debian package name/version list in a controlled, access-restricted location:

```sh
dpkg-query -W -f='${binary:Package}\t${Version}\n'
```

Review package names and versions for disclosure concerns before sharing the output. The current handbook records the observed package count only.
