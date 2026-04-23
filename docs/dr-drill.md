# DR drill — restoring the secondary Console from NFS backup

Operator guide for invoking `playbooks/40-dr-drill.yml` and interpreting
the output. Implements Task 2 from `New-Tasks.md`.

## What this drill does and does not do

**Does:**

- Stops `twistlock.service` on the *secondary* Console.
- Picks the most recent `prisma-*.tar.gz` from the NFS mount at
  `{{ prisma_backup_dir }}` (the same mount the primary writes into).
- Invokes `twistcli restore --backup-file <tarball>` against the local
  Console API to replay the backup.
- Starts `twistlock.service` and waits up to ~5 minutes for
  `/api/v1/_ping` to return 200.
- Writes a timestamped report to `/var/log/prisma-dr/`.

**Does not:**

- Touch the primary Console in any way. The drill is scoped by
  `--limit` to the secondary host name; the `prisma_restore` role's
  interlock aborts if run without an explicit scope.
- Cut over traffic. DNS / load-balancer changes are out of scope for
  Ansible. After a real failover, update them out of band.
- Test data-plane replication of Defender connections. Defenders
  connect to whatever the Console FQDN resolves to; failover is a
  DNS / traffic-steering concern, not a Console-state one.
- Decrypt secrets the backup contains. Secrets round-trip from the
  primary's backup to the secondary's restored DB via the Console's
  built-in encryption; the operator's token only authenticates the
  restore API call.

## Prerequisites

1. **Secondary Console fully installed.** The target host has been
   through Phases 2–10 (baseline, FIPS, storage, podman, TLS, stage,
   config, Phase 9 manual install, ops). `/etc/systemd/system/twistlock.service`
   exists.
2. **NFS mount in place.** `/var/lib/twistlock-backup` is mounted
   (NFS) on the secondary, with read access to the primary's backup
   output. Confirm with `findmnt /var/lib/twistlock-backup` → FSTYPE
   column is `nfs` or `nfs4`.
3. **At least one primary backup landed.** Primary's nightly
   `prisma-backup.timer` has fired at least once and the tarball is
   visible to the secondary via the shared mount.
4. **Console API credentials for the restore call.** Dedicated service
   account, role = *CI Access User* or similar with restore permission.
   Credentials in ansible-vault or Vault; never plaintext.
5. **`twistcli` on the secondary.** Normally provisioned by the
   `prisma_stage` role during Phase 7. Path defaults to
   `{{ prisma_extracted_dir }}/linux/twistcli`.

## Running the drill

Full scheduled drill:

```bash
ansible-playbook playbooks/40-dr-drill.yml \
  --limit prisma-console-dev-dc2 \
  -e prisma_restore_confirmed=true \
  -e prisma_restore_api_user=drill-svc \
  -e @/etc/ansible/drill-secrets.vault.yml \
  --vault-password-file /etc/ansible/vault-pass
```

Pre-flight only (no restore; verifies mount + backup file + twistcli
presence and writes a dry-run report):

```bash
ansible-playbook playbooks/40-dr-drill.yml \
  --limit prisma-console-dev-dc2 \
  -e prisma_restore_confirmed=true \
  -e prisma_restore_dry_run=true
```

Pinned backup file (specific night, not latest):

```bash
ansible-playbook playbooks/40-dr-drill.yml \
  --limit prisma-console-dev-dc2 \
  -e prisma_restore_confirmed=true \
  -e prisma_restore_specific_file=/var/lib/twistlock-backup/prisma-20260410T021513Z.tar.gz
```

## Interpreting the report

Reports land at `/var/log/prisma-dr/dr-drill-<timestamp>.log` on the
secondary. Sample outline:

```
Prisma Console DR Drill Report
==============================
Host:              prisma-console-dev-dc2
Start timestamp:   2026-04-24T02:17:03Z
End timestamp:     2026-04-24T02:19:51Z
Outcome:           success
Dry run:           False

Backup source
-------------
Path:              /var/lib/twistlock-backup/prisma-20260423T021512Z.tar.gz
Size:              142213458 bytes (135.6 MB)
mtime:             2026-04-23T02:15:12Z
NFS source:        /var/lib/twistlock-backup

Console state
-------------
Unit:              twistlock.service
Health URL:        https://localhost:8083/api/v1/_ping
Health status:     pass
Retry budget:      30 x 10s

twistcli invocation
-------------------
Binary:            /opt/prisma-install/.../linux/twistcli
Exit code:         0
```

Key things to check in the Console UI after a **successful** drill:

1. Manage → Defenders — the Defender list should reflect the primary's
   state *as of the backup's mtime*, not the drill time.
2. Manage → Authentication — LDAPS + users present.
3. Monitor → Vulnerabilities → scan history — present.
4. Defend → Policies — intact.

After a **scheduled** drill, reset the secondary to a clean state
before the next primary-write cycle (out-of-band; vary by site —
VMware snapshot revert, re-install, or leave warm).

## Failure modes and triage

| Report line | Likely cause | Next step |
|---|---|---|
| `prisma_restore refused: pass -e prisma_restore_confirmed=true` | Missing interlock flag. | Re-run with `-e prisma_restore_confirmed=true`. |
| `findmnt … not NFS` | Local dir shadowing the NFS mount. | `umount /var/lib/twistlock-backup; mount -a`; verify `/etc/fstab`. |
| `No backup tarball matched …` | Primary hasn't pushed, or the glob is wrong. | On primary, `systemctl status prisma-backup.timer`; run manually with `systemctl start prisma-backup.service`. |
| `twistcli not found at …` | `prisma_stage` not run on secondary, or custom path. | Run the console-path playbooks against the secondary, or set `prisma_restore_twistcli` to the actual path. |
| `restore ran but health check failed` | Console failed to come up post-restore. | `journalctl -u twistlock.service -n 200 --no-pager`; `podman logs twistlock_console` for Console-side errors; check for TLS cert mismatch between primary (baked into backup) and secondary host. |
| `twistcli restore` exit code ≠ 0 | API auth, flag mismatch, or backup corruption. | Re-read `restore_run.stderr` in the report. If flag mismatch, run `twistcli restore --help` on the secondary and compare against the role's defaults (see `roles/prisma_restore/README.md`). |

## Non-destructive enforcement

The role's safety interlock (`prisma_restore_confirmed: false` by
default) exists because `twistcli restore` rewrites the Console's
database. Running it against the *primary* would wipe the live state
in favour of the backup. The three operational guarantees:

1. **Explicit `-e prisma_restore_confirmed=true` required per run.**
   No default override; every drill invocation has this flag on the
   command line.
2. **`--limit` required.** The playbook targets `prisma_console`
   broadly; without `--limit`, both primary and secondary would be
   in scope. Operator discipline; there is no Ansible-native "refuse
   if more than one host matched" guard. Document this in the change
   ticket for every drill.
3. **NFS-mount assert.** A secondary whose mount is not NFS fails
   fast — either the mount dropped (real problem) or the wrong host
   is being targeted.

## Running the drill on a schedule

Not automated by Ansible — the drill is **operator-invoked** because
its output is meant to be reviewed each run. A reasonable cadence is
quarterly, tracked as a change ticket with the report log attached.
If site policy requires automated drills, wrap the `ansible-playbook`
command above in a systemd timer on an ops host (not the secondary
Console itself); alerting on any non-zero exit code is sufficient.

## Cross-references

- [`roles/prisma_restore/README.md`](../roles/prisma_restore/README.md) — role vars and defaults.
- [`roles/prisma_backup/README.md`](../roles/prisma_backup/README.md) — nightly push side.
- [`docs/runbook.md`](runbook.md) § 9 — backup/restore context.
- [`docs/manual-validation-guide.md`](manual-validation-guide.md) —
  extend with `twistcli restore --help` capture when lab access
  becomes available, to confirm the flag shape in
  `prisma_restore/tasks/main.yml`.
