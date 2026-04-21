# prisma_monitoring

## Purpose

Stages and SHA-256 verifies the operator-supplied `node_exporter` tarball,
installs it as a systemd unit, opens `node_exporter_port` in firewalld, and
drops a rsyslog conf that ships Prisma Console logs to the site SIEM. Runs
last in Phase 10 (`playbooks/30-prisma-ops.yml`), after the Console service
and backup flow are healthy.

## Variables

From `roles/prisma_monitoring/defaults/main.yml`:

| Variable | Default | Description |
|---|---|---|
| `node_exporter_user` | `node_exporter` | System user the unit runs as. |
| `node_exporter_bin` | `/usr/local/bin/node_exporter` | Installed binary path. |
| `node_exporter_tarball` | `node_exporter-{{ node_exporter_version }}.linux-amd64.tar.gz` | Tarball filename under role `files/` (operator-supplied). |
| `node_exporter_sha256` | `""` | **Required.** 64-char SHA-256 hex of the tarball. Empty or mismatching digest aborts the role. |

Also consumes, from `group_vars/prisma_console.yml`:
`node_exporter_version`, `node_exporter_port`, `siem_server`, `siem_port`.

## Dependencies

- **Ansible collections:** `ansible.posix` (firewalld module).
- **Role order:** must run after `rhel_baseline` (rsyslog + firewalld
  running) and after `prisma_systemd` (Console unit is logging).
- **Operator artefacts:**
  - `node_exporter-<ver>.linux-amd64.tar.gz` dropped under the play's
    `files/` search path (alongside the Prisma tarball).
  - Matching SHA-256 digest supplied via `node_exporter_sha256` — generate
    locally with `sha256sum node_exporter-<ver>.linux-amd64.tar.gz`.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  vars:
    node_exporter_version: "1.8.2"
    node_exporter_sha256: "6809dd0b3ec45fd6e992c4ee6e3f2f4d4e9f2a0e5c1b8a2a1c0d0a0a0a0a0a0a"
  roles:
    - role: prisma_monitoring
      tags: [monitoring]
```

## Tags exposed

Invoked with `tags: [monitoring]` in `playbooks/30-prisma-ops.yml`.

```bash
ansible-playbook playbooks/30-prisma-ops.yml --tags monitoring
```
