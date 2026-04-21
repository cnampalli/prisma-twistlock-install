# prisma_systemd

## Purpose

Drops a hardening override at
`/etc/systemd/system/{{ prisma_systemd_unit }}.service.d/override.conf`
(raises `TasksMax`, sets `RestartSec`, enables cgroup accounting) on top of
the `twistlock.service` unit the Prisma installer created during Phase 9.
The role warns-and-skips if the unit is missing — run the installer first.
Runs first in Phase 10 (`playbooks/30-prisma-ops.yml`), before
`prisma_logrotate`, `prisma_backup`, and `prisma_monitoring`.

## Variables

From `roles/prisma_systemd/defaults/main.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_systemd_unit` | `twistlock` | Name of the Prisma-created systemd unit; the role overrides rather than replaces it. |
| `prisma_systemd_tasks_max` | `8192` | `TasksMax=` for the override. Default 4096 blocks new goroutines under heavy Defender fleets. |
| `prisma_systemd_restart_sec` | `10` | `RestartSec=` so the DB has time to flush before a restart. |

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** must run after the operator has manually executed Phase 9
  (`./twistlock.sh -s console`) so `/etc/systemd/system/twistlock.service`
  exists. Must run before `prisma_backup` (backup API needs a running unit)
  and `prisma_monitoring` (Console must be emitting logs).
- **Operator artefacts:** none beyond the installer-created systemd unit.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  roles:
    - role: prisma_systemd
      tags: [systemd]
```

## Tags exposed

Invoked with `tags: [systemd]` in `playbooks/30-prisma-ops.yml`.

```bash
ansible-playbook playbooks/30-prisma-ops.yml --tags systemd
```
