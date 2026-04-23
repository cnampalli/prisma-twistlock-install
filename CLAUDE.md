# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Ansible automation for a **greenfield, air-gapped** install of Prisma Cloud Compute Edition Console v34.01.126 on RHEL 8.10, rootful podman, FIPS 140-2 end-to-end.

Source of truth: [`docs/runbook.md`](docs/runbook.md). Each step is tagged `[A]` (automated here), `[A/M]` (automated + manual verify), or `[M]` (stays manual — e.g. Phase 9's interactive `./twistlock.sh`). **Do not wrap `[M]` steps in Ansible.** Confirm the runbook tag before automating anything new.

## Commands

```bash
# Lint (matches CI)
yamllint . && ansible-lint && ansible-playbook --syntax-check playbooks/site.yml

# Pinned collection deps (requirements.yml is exact-pinned on purpose)
ansible-galaxy collection install -r requirements.yml

# Full build (Phases 2–8). Phase 9 manual; then run 30-prisma-ops.yml.
ansible-playbook -i inventory/dev/hosts.yml playbooks/site.yml
# Slice by tag: baseline, fips, storage, podman, tls, stage, config,
# systemd, logrotate, backup, monitoring (e.g. --tags podman)

# Lab verification (RHEL 8.10 sandbox; asserts [A] acceptance criteria)
cd molecule/default && molecule test
molecule converge -s default && molecule verify -s default  # dev loop
```

## Architecture

Playbooks map 1:1 to runbook phases and compose roles in order:

- `00-baseline.yml` — two plays: `rhel_baseline` + `fips_enable` on `prisma_all`, `prisma_storage` on `prisma_console` (Phases 2–4).
- `10-podman.yml` → `podman_config` on `prisma_console` (Phase 5).
- `11-docker.yml` → `docker_stage` → `docker_config` on `prisma_scanner:prisma_sandbox` (Phase 3b).
- `20-prisma-stage.yml` → `prisma_tls` → `prisma_stage` → `prisma_config` on `prisma_console` (Phases 6–8). Ends with a reminder to run `./twistlock.sh -s console` manually (Phase 9).
- `30-prisma-ops.yml` → `prisma_systemd` → `prisma_logrotate` → `prisma_backup` → `prisma_monitoring` on `prisma_console` (Phase 10). Run **after** Phase 9 has created `/etc/systemd/system/twistlock.service`.
- `40-dr-drill.yml` → `prisma_restore` — operator-invoked DR drill against a secondary. Requires `--limit <secondary-host>` and `-e prisma_restore_confirmed=true`. See `docs/dr-drill.md`.
- `site.yml` imports 00 → 10 → 11 → 20 (not 30 — depends on manual Phase 9; not 40 — operator-scoped).

**docker / podman mutex**: `prisma_console` runs rootful podman; `prisma_scanner` and `prisma_sandbox` run docker-ce. Never on the same host. Podman lives in `rhel_baseline_container_packages` (set to `[podman]` in `group_vars/prisma_console.yml` only). `docker_stage` asserts `rpm -q podman` fails before it installs anything.

## Invariants — do not break

- **Air-gapped**: no runtime fetches from the public internet. Tarballs, certs, binaries are operator-supplied under `files/` (gitignored).
- **FIPS**: `prisma_fips_enabled: true` is load-bearing; disabling skips `fips_enable` and fails validation.
- **Secrets** (licence, LDAPS bind pw, backup token/URL) come from Vault or ansible-vault — never plain YAML. `group_vars/prisma_console.yml` is non-secret defaults only.
- **Collections exact-pinned** in `requirements.yml` — patch bumps have broken things. Bump intentionally and test in `molecule/full`.
- **Lint skips** in `.ansible-lint` are justified in-line — read before removing or adding.
