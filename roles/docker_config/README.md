# docker_config

## Purpose

Configures the docker-ce daemon on scanner / sandbox hosts: writes
`/etc/docker/daemon.json` (overlay2 storage, journald logging, ulimits,
live-restore, systemd cgroup driver) and a matching
`/etc/systemd/system/docker.service.d/override.conf` (accounting +
`TasksMax` + restart policy). Enables and starts `docker.service`, then
smoke-tests with `docker info`. Memory-limit knobs are intentionally
absent here; they land in Phase 5 alongside the Console's systemd override.

## Variables

| Variable | Default | Description |
|---|---|---|
| `docker_storage_driver` | `overlay2` | daemon.json `storage-driver`. |
| `docker_log_driver` | `journald` | daemon.json `log-driver`. |
| `docker_log_size_max` | `100m` | Per-log max size. |
| `docker_log_max_file` | `3` | Log rotation count. |
| `docker_nofile_soft` / `docker_nofile_hard` | `65535` | File descriptor ulimits. |
| `docker_nproc_soft` / `docker_nproc_hard` | `4096` | Process ulimits. |
| `docker_live_restore` | `true` | Survive daemon restarts. |
| `docker_systemd_tasks_max` | `8192` | systemd `TasksMax=`. |
| `docker_systemd_restart_sec` | `5` | systemd `RestartSec=`. |

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** runs after `docker_stage`.
- Operator artefacts: none.

## Example usage

```yaml
- hosts: prisma_scanner:prisma_sandbox
  become: true
  roles:
    - role: docker_config
      tags: [docker_config]
```

## Tags exposed

Invoked with `tags: [docker_config]` in `playbooks/11-docker.yml`.
