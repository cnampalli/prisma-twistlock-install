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
| `rhel_baseline_packages` | `[podman, lvm2, xfsprogs, chrony, firewalld, policycoreutils-python-utils, openssl, jq, tar, gzip, logrotate, rsyslog, audit, curl]` | Minimum package set; supports podman, LVM/XFS, SELinux modules, and the backup script. |
| `rhel_baseline_firewall_ports` | `["{{ prisma_https_port }}/tcp", "{{ prisma_comm_port }}/tcp"]` | Ports opened permanently in firewalld. |
| `rhel_baseline_firewall_services` | `[ssh]` | firewalld service names to allow. |
| `rhel_baseline_sysctl` | `{vm.swappiness: 10, vm.max_map_count: 262144, fs.file-max: 2097152, net.core.somaxconn: 1024, net.ipv4.tcp_max_syn_backlog: 1024}` | Kernel tunables required by Prisma Console under load. |
| `rhel_baseline_journald_max_use` | `2G` | `SystemMaxUse=` for journald. |
| `rhel_baseline_journald_keep_free` | `1G` | `SystemKeepFree=` for journald. |

Also consumes, from `group_vars/prisma_console.yml`: `prisma_https_port`,
`prisma_comm_port`, `ntp_servers`.

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
