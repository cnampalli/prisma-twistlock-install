# prisma-twistlock-install

Greenfield, air-gapped installation of **Prisma Cloud Compute Edition Console v34.01.126** on RHEL 8.10 with rootful podman, FIPS 140-2 mode enabled end-to-end.

The canonical runbook is [`docs/runbook.md`](docs/runbook.md). This repo provides the Ansible automation for every step the runbook tags `[A]` (automate) or `[A/M]` (automate with manual verification). Steps tagged `[M]` (manual) remain manual by design — interactive installers, first-run wizards, or one-time compliance-load-bearing decisions.

## Layout

```
.
├── README.md                 # this file
├── docs/runbook.md           # full runbook (Manual vs Ansible tagged per step)
├── ansible.cfg
├── inventory/hosts.yml       # inventory template (populate at runtime)
├── group_vars/
│   └── prisma_console.yml    # non-secret defaults
├── host_vars/
│   └── prisma-console-01.yml # example host override
├── playbooks/
│   ├── site.yml              # run everything, ordered
│   ├── 00-baseline.yml       # Phases 2–4
│   ├── 10-podman.yml         # Phase 5
│   ├── 20-prisma-stage.yml   # Phases 6–8
│   └── 30-prisma-ops.yml     # Phase 10
├── roles/
│   ├── rhel_baseline/
│   ├── fips_enable/
│   ├── prisma_storage/
│   ├── podman_config/
│   ├── prisma_tls/
│   ├── prisma_stage/
│   ├── prisma_config/
│   ├── prisma_systemd/
│   ├── prisma_logrotate/
│   ├── prisma_backup/
│   └── prisma_monitoring/
├── files/                    # operator-supplied drop-in artefacts (tarball, certs) — gitignored
└── molecule/default/         # lab scenario: converge + verify
```

## Pre-flight

Operator supplies (NOT committed; see `.gitignore`):

- `files/prisma_cloud_compute_edition_34_01_126.tar.gz`
- `files/prisma_cloud_compute_edition_34_01_126.tar.gz.sha256` (single line: `<sha256>  prisma_cloud_compute_edition_34_01_126.tar.gz`)
- `files/console.crt`, `files/console.key`, `files/ca-chain.crt`

Secrets (licence key, LDAPS bind password, Prisma backup token, internal backup URL) come from Vault via the same AppRole pattern as the `twistlock-exemption` repo, or from an Ansible Vault-encrypted file — never from plain YAML.

## Run

```bash
# Full build (matches runbook Phases 2–10)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml

# Slice — re-run only podman configuration
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags podman

# Slice — re-run only the operational layer (systemd/logrotate/backup/monitoring)
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --tags systemd,logrotate,backup,monitoring
```

Phase 9 (running `./twistlock.sh -s console`) is deliberately **not** wrapped — see runbook §Phase 9.

## Tags

`baseline`, `fips`, `storage`, `podman`, `tls`, `stage`, `config`, `systemd`, `logrotate`, `backup`, `monitoring`.

## Verify

A Molecule scenario under `molecule/default/` converges a RHEL 8.10 sandbox and asserts every `[A]` acceptance criterion from the runbook. Run before touching production:

```bash
cd molecule/default && molecule test
```
