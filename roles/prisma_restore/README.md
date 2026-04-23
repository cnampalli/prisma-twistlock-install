# prisma_restore

## Purpose

DR drill role. Restores the **secondary** Prisma Console from a backup
tarball sitting on the shared NFS mount at `prisma_backup_dir`, verifies
Console health post-restore, and writes a timestamped report. Invoked
by `playbooks/40-dr-drill.yml`. Non-destructive to the primary —
primary and secondary are separate hosts; the drill only touches the
host the playbook is targeted at.

## Safety interlock

The role refuses to run unless `prisma_restore_confirmed=true` is set
at runtime. This is an operator affirmation that:

1. The target host is the **secondary**, not the primary.
2. The play invocation uses `--limit` to scope to that host.
3. Restoring the Console's database is acceptable at this time.

There is **no default that bypasses this interlock**; always pass
`-e prisma_restore_confirmed=true` explicitly.

## Variables

From `roles/prisma_restore/defaults/main.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_restore_nfs_path` | `{{ prisma_backup_dir }}` | Directory on the NFS mount to scan for backups. Must match the path the primary writes into. |
| `prisma_restore_tarball_glob` | `prisma-*.tar.gz` | Glob for the primary's nightly output. |
| `prisma_restore_specific_file` | `""` | Pin a specific file path instead of "latest". |
| `prisma_restore_confirmed` | `false` | Safety interlock — must be `true` at runtime. |
| `prisma_restore_dry_run` | `false` | Run pre-flight only; skip the stop / restore / start / verify sequence. |
| `prisma_restore_twistcli` | `{{ prisma_extracted_dir }}/linux/twistcli` | Path to twistcli on the secondary. |
| `prisma_restore_unit` | `twistlock` | Systemd unit name for the local Console. |
| `prisma_restore_health_url` | `https://localhost:{{ prisma_https_port }}/api/v1/_ping` | Post-restore health endpoint. |
| `prisma_restore_health_retries` | `30` | Health-check retry count. |
| `prisma_restore_health_delay` | `10` | Health-check retry delay in seconds. |
| `prisma_restore_report_dir` | `/var/log/prisma-dr` | Where drill reports land. |
| `prisma_restore_api_user` | `""` | Console API user (env var to twistcli). Operator supplies via `-e` or `host_vars`. |
| `prisma_restore_api_password` | `""` | Console API password. Vault-only; never plaintext. |

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** runs standalone via `playbooks/40-dr-drill.yml`.
  Depends on `prisma_stage` having staged `twistcli` on the secondary
  (or a manual copy under `prisma_restore_twistcli`).
- **Filesystem:** the path at `prisma_restore_nfs_path` must be an
  NFS mount. The role asserts this via `findmnt` and aborts if the
  mount is local — a local empty dir would silently restore nothing.
- **Phase 9 completed:** the target host must have been through a
  full manual Phase 9 install, so `/etc/systemd/system/twistlock.service`
  exists and can be stop/started.

## Twistcli flag shape — pending manual validation

`twistcli restore` is confirmed to exist as a top-level subcommand
(see `docs/fips-feasibility.md`). The exact flag form for restoring a
local tarball is **best-guess** in this role:

```
twistcli restore --address <url> --backup-file <path>
```

with `PRISMA_USER` / `PRISMA_PASSWORD` in the environment. If
`twistcli restore --help` on the extracted tarball shows different
flags, either patch the `Run twistcli restore` task or override via
a custom playbook `vars:` block. Running `twistcli restore --help`
should be part of the extended Phase 3 validation checklist in
`docs/manual-validation-guide.md`.

## Example usage

```bash
# Scoped to the secondary host; interlock explicitly set.
ansible-playbook playbooks/40-dr-drill.yml \
  --limit prisma-console-dev-dc2 \
  -e prisma_restore_confirmed=true \
  -e prisma_restore_api_user=drill \
  -e prisma_restore_api_password=@/root/.drill-token \
  --vault-password-file /etc/ansible/vault-pass

# Dry-run — verify pre-flight + backup selection but skip restore.
ansible-playbook playbooks/40-dr-drill.yml \
  --limit prisma-console-dev-dc2 \
  -e prisma_restore_confirmed=true \
  -e prisma_restore_dry_run=true
```

## Tags exposed

Invoked with `tags: [restore, dr]` in `playbooks/40-dr-drill.yml`.
