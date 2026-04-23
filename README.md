# prisma-twistlock-install

Greenfield, air-gapped installation of **Prisma Cloud Compute Edition Console v34.01.126** on RHEL 8.10 with rootful podman, FIPS 140-2 mode enabled end-to-end.

The canonical runbook is [`docs/runbook.md`](docs/runbook.md). This repo provides the Ansible automation for every step the runbook tags `[A]` (automate) or `[A/M]` (automate with manual verification). Steps tagged `[M]` (manual) remain manual by design — interactive installers, first-run wizards, or one-time compliance-load-bearing decisions.

## Layout

```
.
├── README.md                 # this file
├── docs/runbook.md           # full runbook (Manual vs Ansible tagged per step)
├── ansible.cfg
├── inventory/                # directory-per-env (dev populated; others are templates)
│   ├── dev/
│   │   ├── hosts.yml         # 2 DCs × 3 VM roles (console, scanner, sandbox)
│   │   ├── group_vars/       # env-wide + per-DC overrides (all.yml, dc1.yml, dc2.yml)
│   │   └── host_vars/        # per-host overrides
│   ├── pre-prod-ht/hosts.yml # commented template
│   ├── pre-prod-vht/hosts.yml
│   ├── prod-ht/hosts.yml
│   └── prod-vht/hosts.yml
├── group_vars/               # cross-env defaults (applied to every env)
│   ├── all.yml               # fleet-wide
│   ├── prisma_console.yml
│   ├── prisma_scanner.yml
│   └── prisma_sandbox.yml
├── playbooks/
│   ├── site.yml              # run everything, ordered (console path)
│   ├── 00-baseline.yml       # Phases 2–4 (console)
│   ├── 10-podman.yml         # Phase 5 (console)
│   ├── 11-docker.yml         # docker-ce for scanner + sandbox (standalone)
│   ├── 20-prisma-stage.yml   # Phases 6–8 (console)
│   └── 30-prisma-ops.yml     # Phase 10 (console)
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
│   ├── prisma_monitoring/
│   ├── docker_stage/          # scanner + sandbox: stage + install docker-ce RPMs
│   └── docker_config/         # scanner + sandbox: daemon.json + systemd override
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
ansible-playbook -i inventory/dev/hosts.yml playbooks/site.yml

# Slice — re-run only podman configuration
ansible-playbook -i inventory/dev/hosts.yml playbooks/site.yml --tags podman

# Slice — re-run only the operational layer (systemd/logrotate/backup/monitoring)
ansible-playbook -i inventory/dev/hosts.yml playbooks/site.yml --tags systemd,logrotate,backup,monitoring
```

Phase 9 (running `./twistlock.sh -s console`) is deliberately **not** wrapped — see runbook §Phase 9.

## Tags

`baseline`, `fips`, `storage`, `podman`, `tls`, `stage`, `config`, `systemd`, `logrotate`, `backup`, `monitoring`.

## Verify

A Molecule scenario under `molecule/default/` converges a RHEL 8.10 sandbox and asserts every `[A]` acceptance criterion from the runbook. Run before touching production:

```bash
cd molecule/default && molecule test
```

## Documentation

| Doc | Purpose |
|---|---|
| [`docs/runbook.md`](docs/runbook.md) | Canonical runbook for the Console install. Source of truth. |
| [`docs/implementation-plan.md`](docs/implementation-plan.md) | Multi-env + scanner + sandbox + DR + memory roadmap. Phased rollout. |
| [`docs/fips-feasibility.md`](docs/fips-feasibility.md) | Kernel / docker-ce / Palo Alto userland FIPS analysis. Known findings + open questions. |
| [`docs/manual-validation-guide.md`](docs/manual-validation-guide.md) | Phase 3 lab-VM procedure: does `twistcli defender install standalone` hardcode docker? |
| [`docs/manual-install/console.md`](docs/manual-install/console.md) | Operator script for building the Console without Ansible. |
| [`docs/manual-install/image-scanner.md`](docs/manual-install/image-scanner.md) | Operator script for the standalone-Defender scanner VM. |
| [`docs/manual-install/sandbox.md`](docs/manual-install/sandbox.md) | Operator script for the twistcli sandbox / CI-agent VM. |
