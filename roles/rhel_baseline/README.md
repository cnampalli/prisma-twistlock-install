# rhel_baseline

## Purpose

Brings a fresh RHEL 8 host to the hardening baseline required by Prisma
Console: asserts distro, installs the baseline package set, enforces
SELinux, enables `chronyd`/`firewalld`/`auditd`/`rsyslog`, renders
`chrony.conf`, hardens `sshd`, opens Console/Defender ports in firewalld,
applies a sysctl drop-in, and caps journald growth. Runs first in Phase 2
(`playbooks/00-baseline.yml`), before `fips_enable` and `prisma_storage`.

## Variables

From `roles/rhel_baseline/defaults/main.yml`:

| Variable | Default | Description |
|---|---|---|
| `rhel_baseline_packages` | `[lvm2, xfsprogs, chrony, firewalld, policycoreutils-python-utils, openssl, jq, tar, gzip, logrotate, rsyslog, audit, curl]` | Fleet-wide package set (every Prisma host class). Does **not** include podman — see `rhel_baseline_container_packages`. |
| `rhel_baseline_container_packages` | `[]` | Host-class-specific container runtime package. Set to `[podman]` in `group_vars/prisma_console.yml`; scanner + sandbox leave it empty (docker-ce is installed by `docker_stage`). |
| `rhel_baseline_firewall_ports` | `[]` | Ports opened permanently in firewalld. Empty by default — Console-specific ports (`8083/tcp`, `8084/tcp`) are set in `group_vars/prisma_console.yml`. |
| `rhel_baseline_firewall_services` | `[ssh]` | firewalld service names to allow. |
| `rhel_baseline_sysctl` | 8 keys — `vm.swappiness: 10`, `vm.overcommit_memory: 1`, `vm.dirty_ratio: 15`, `vm.dirty_background_ratio: 5`, `vm.max_map_count: 262144`, `fs.file-max: 2097152`, `net.core.somaxconn: 1024`, `net.ipv4.tcp_max_syn_backlog: 1024` | Kernel tunables required by Prisma Console under load; memory knobs are load-bearing for Task 3's 95%-RAM target. |
| `rhel_baseline_disable_thp` | `true` | Install + enable a systemd oneshot that writes `never` to `/sys/kernel/mm/transparent_hugepage/{enabled,defrag}`. Required by MongoDB (embedded in Console). |
| `rhel_baseline_journald_max_use` | `2G` | `SystemMaxUse=` for journald. |
| `rhel_baseline_journald_keep_free` | `1G` | `SystemKeepFree=` for journald. |
| `rhel_baseline_password_auth_exception` | `false` | Opt-in. The baseline disables SSH password auth globally (`PasswordAuthentication no`). When `true`, a `Match`-block drop-in (`/etc/ssh/sshd_config.d/50-prisma-admin-password.conf`) re-enables password auth **only** for the admin source network below — for jump-host/MobaXterm admin access with AD credentials. Loosens the baseline; keys/Kerberos preferred where feasible. |
| `rhel_baseline_password_auth_cidr` | `""` | Admin source network (CIDR) the password exception applies to, e.g. `10.0.0.0/8`. **Required** when the exception is enabled; scope it as tightly as you can. |
| `rhel_baseline_password_auth_group` | `""` | Optional — further restrict the exception to a group (`Match … Group`). |

Also consumes:
- From `group_vars/prisma_console.yml`: `prisma_https_port`, `prisma_comm_port`
  (Console-only — drives the `rhel_baseline_firewall_ports` override).
- From `group_vars/prisma_all.yml`: `ntp_servers` — **required, non-empty**.
  Lives at the prisma_all scope so every host class (console, scanner,
  sandbox) inherits the same NTP peers; the role asserts the list is
  non-empty before rendering `/etc/chrony.conf` to prevent a silent
  empty-server config that lets chronyd start without ever syncing.

## Dependencies

- **Ansible collections:** `ansible.posix` — `selinux`, `firewalld` modules.
- **Role order:** runs first in the install pipeline. Required before
  `fips_enable` (needs `crypto-policies-scripts`, pulled in as a transitive
  dep of the baseline packages) and before `prisma_storage` (`lvm2`,
  `xfsprogs`, `policycoreutils-python-utils`).
- **Operator artefacts:** none — all packages come from the subscribed RHEL
  repos (or the air-gapped Satellite mirror).

## Example usage

```yaml
- hosts: prisma_console
  become: true
  roles:
    - role: rhel_baseline
      tags: [baseline]
```

## Tags exposed

Invoked with `tags: [baseline]` in `playbooks/00-baseline.yml`.

```bash
ansible-playbook playbooks/00-baseline.yml --tags baseline
```
