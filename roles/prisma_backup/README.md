# prisma_backup

## Purpose

Installs the nightly Prisma Console backup flow — a curl-based script that
hits the Console backup API and uploads the resulting archive to an internal
artefact store, driven by a systemd service + timer. Runs in Phase 10
(`playbooks/30-prisma-ops.yml`), after the installer has been run manually
and `prisma_systemd` has brought the Console unit up. TLS verification of
the Console/artefact endpoints is gated by `prisma_backup_insecure_tls`
(default `false`).

## Variables

From `roles/prisma_backup/defaults/main.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_backup_schedule` | `"*-*-* 02:15:00"` | systemd `OnCalendar` expression for the timer. |
| `prisma_backup_jitter_sec` | `300` | `RandomizedDelaySec` applied to the timer. |
| `prisma_backup_script` | `/usr/local/sbin/prisma-backup.sh` | Rendered script path. |
| `prisma_backup_env_file` | `/etc/prisma/backup.env` | EnvironmentFile consumed by the service; contains operator-supplied secrets. |
| `prisma_backup_insecure_tls` | `false` | Disable curl TLS verification. Keep `false` in production. |
| `prisma_backup_ca_bundle` | `{{ prisma_cert_ca_path }}` | CA bundle passed as `--cacert` when TLS verify is on. |

Also consumes, from `group_vars/prisma_console.yml`: `prisma_backup_dir`,
`prisma_backup_retention_days`, `prisma_backup_user`, `prisma_https_port`,
and the Vault-supplied `prisma_backup_token` + `internal_backup_url`.

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** must run after `prisma_systemd` (Console service is up and
  the backup API responds) and after `prisma_tls` (CA bundle at
  `prisma_cert_ca_path` exists).
- **Operator artefacts:** Vault-supplied `prisma_backup_token` (Console
  service-account token) and `internal_backup_url` (artefact store PUT URL).
  Do not commit real values.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  vars:
    prisma_backup_token: "{{ lookup('community.hashi_vault.hashi_vault', 'secret=kv/prisma:token') }}"
    internal_backup_url: https://artefacts.internal/prisma/
  roles:
    - role: prisma_backup
      tags: [backup]
```

## Tags exposed

Invoked with `tags: [backup]` in `playbooks/30-prisma-ops.yml`.

```bash
ansible-playbook playbooks/30-prisma-ops.yml --tags backup
```
