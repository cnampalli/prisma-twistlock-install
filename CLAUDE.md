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
# Pick the env: inventories/{dev,preproduction,production}/hosts.yml
ansible-playbook -i inventories/dev/hosts.yml playbooks/site.yml
# Slice by tag: baseline, fips, storage, podman, tls, stage, config,
# systemd, logrotate, backup, monitoring (e.g. --tags podman)

# Lab verification (RHEL 8.10 sandbox; asserts [A] acceptance criteria)
cd molecule/default && molecule test
molecule converge -s default && molecule verify -s default  # dev loop

# Controller-as-code (AAP 2.4) — control-node only, separate from the EE.
ansible-galaxy collection install -r controller/requirements.controller.yml
ansible-playbook -i controller/inventory.ini controller/configure.yml --syntax-check
```

## Architecture

Playbooks map 1:1 to runbook phases and compose roles in order:

- `00-baseline.yml` — two plays: `rhel_baseline` + `fips_enable` on `prisma_all`, `prisma_storage` on `prisma_console` (Phases 2–4).
- `10-podman.yml` → `podman_config` on `prisma_console` (Phase 5).
- `11-docker.yml` → `docker_stage` → `docker_config` on `prisma_scanner:prisma_sandbox` (Phase 3b).
- `20-prisma-stage.yml` → `prisma_tls` → `prisma_stage` → `prisma_config` on `prisma_console` (Phases 6–8). Ends with a reminder to run `./twistlock.sh -s console` manually (Phase 9).
- `30-prisma-ops.yml` → `prisma_systemd` → `prisma_logrotate` → `prisma_backup` → `prisma_monitoring` on `prisma_console` (Phase 10). Run **after** Phase 9 has created `/etc/systemd/system/twistlock.service`.
- `40-dr-drill.yml` → `prisma_restore` — operator-invoked DR drill. Targets `prisma_console:&secondary` so it self-limits to secondary consoles even without `--limit`. Requires `-e prisma_restore_confirmed=true`. See `docs/dr-drill.md`.
- `site.yml` imports 00 → 10 → 11 → 20 (not 30 — depends on manual Phase 9; not 40 — operator-scoped).

**docker / podman mutex**: `prisma_console` runs rootful podman; `prisma_scanner` and `prisma_sandbox` run docker-ce. Never on the same host. Podman lives in `rhel_baseline_container_packages` (set to `[podman]` in `inventories/<env>/group_vars/prisma_console.yml` only). `docker_stage` asserts `rpm -q podman` fails before it installs anything.

**Inventory layout**: `inventories/<env>/` for `dev`, `preproduction`, `production`. Each host sits on two axes — a role group (`prisma_console`/`prisma_scanner`/`prisma_sandbox` under `prisma_all`) and a **site group** (`primary` = live, `secondary` = DR target). Hosts are named `prisma-<role>-<env>-<site>`. Site group_vars (`{primary,secondary}.yml`) set `ansible_group_priority: 10` so site vars beat role-class vars (bare names like `primary` sort below `prisma_*` and would otherwise lose the tie). `secondary` carries `is_dr_target: true`.

**AAP / Controller-as-code**: `controller/` holds the Automation Controller 4.5.x (AAP 2.4) desired state (project, 3 inventories, credential types, job templates, two workflows `prisma-build`/`prisma-ops`, DR survey) as data for `infra.controller_configuration.dispatch`. The EE is assumed to pre-exist. Its collections live in `controller/requirements.controller.yml` — **never** add them to the air-gapped `requirements.yml`. See `controller/README.md`.

## Invariants — do not break

- **Air-gapped**: no runtime fetches from the public internet. Tarballs, certs, binaries are operator-supplied under `files/` (gitignored).
- **FIPS**: `prisma_fips_enabled: true` is load-bearing; disabling skips `fips_enable` and fails validation.
- **Secrets** (backup token/URL, restore API creds) are injected at job launch by **AAP Controller credentials** (`controller/group_vars/all/controller_credential_types.yml`) — never committed. Licence + LDAPS bind pw are Phase 11 manual-UI items no playbook reads. `inventories/<env>/group_vars/prisma_console.yml` is non-secret defaults only. Local CLI runs pass secrets via `-e @local-secrets.yml` (gitignored).
- **Collections exact-pinned** in `requirements.yml` — patch bumps have broken things. Bump intentionally and test in `molecule/full`.
- **Lint skips** in `.ansible-lint` are justified in-line — read before removing or adding.
