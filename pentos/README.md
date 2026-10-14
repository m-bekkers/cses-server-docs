# Pentos System Documentation

This directory is the operational handbook for the Linux host `pentos.engsoc.net`. It is a point-in-time record of observations made on 2026-10-13, supplemented by deployment artifacts in `/home/bekkers/working/`. Repository configuration is evidence of intended or documented deployment behavior, not proof of live state.

## Documents

- [System profile](system-profile.md): host identity, compute, storage, network, and software baseline.
- [Service catalog](service-catalog.md): observed system services, listening sockets, and applications documented in the working tree.
- [Application architecture](application-architecture.md): repository-defined portal and OpenProject components, request paths, and deployment boundaries.
- [Operations and recovery](operations.md): routine checks, safe response guidance, and information that still needs confirmation.

## Evidence and maintenance

The words **Observed**, **Repository-defined**, and **Unverified** distinguish direct host evidence, checked-in configuration/documentation, and facts that still require an authorized live check. Treat dynamic observations as a snapshot, not as a current-state guarantee. Record the capture date, source, and verification status when refreshing a value. Review this handbook after host, network, identity, storage, or deployment changes, and at least annually. The service owner and formal review cadence have not been identified in the available evidence.

The documentation was checked against the deployment checkouts under `/home/bekkers/working/` on 2026-10-13. No privileged host refresh was performed as part of this documentation review.

## Scope and handling

The inventory describes visible host configuration and status; it is not a full forensic image, configuration backup, or guarantee that every application is represented. Dynamic values such as addresses, package versions, ports, service state, and disk utilization can change. Re-check them before operational use.

No credentials, private keys, environment-file contents, machine IDs, or serial numbers are recorded here. Keep this handbook and any future generated inventories access-controlled because hostnames, addresses, service exposure, and deployment paths are operationally sensitive. Never copy secret values into this directory.

## Known limitations and urgent observations

- Root filesystem `/` was 96% used, with about 40 GiB available. Review capacity promptly; do not remove Docker volumes or application data without confirming ownership and backups.
- `md0` was a healthy two-member RAID1 array at capture time (`[UU]`). RAID is not a backup.
- Several sockets bind to all interfaces, including TCP 5432, 80, 22, and 2222. Binding does not prove internet reachability; firewall and upstream ACL policy were not inspected.
- Six SSSD responder units were failed at capture time. The main SSSD service was running.
- Docker status could not be queried by the current account; the Docker socket is owned by `root:docker`, and non-interactive sudo requires a password. Container state and port ownership therefore remain unverified.
- The checked-in OpenProject Compose manifest contains insecure placeholder defaults. Confirm the deployed configuration overrides them; do not rely on those defaults.
- Repository deployment records disagree on the static site's document root: the deployment README names `/var/www/tasks.mycses.ca`, while the retained Compose override mounts `/var/www/tasks`. Confirm the actual host path and active Compose files before changing or redeploying the site.

## Refreshing this handbook

Run the read-only checks in [Operations and recovery](operations.md) after significant hardware, network, deployment, or OS changes. Update the capture date and distinguish direct observations from repository-derived details. Use an approved privileged account for firewall, Docker, and root-owned process inspection, and record findings without recording secret values.
