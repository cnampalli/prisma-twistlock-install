# Controller-as-Code (AAP 2.4 / Automation Controller 4.5.x)

Declarative definitions for the Automation Controller objects that run the
Prisma Console install. The data lives under `group_vars/all/` and is applied by
`configure.yml`, which calls `infra.controller_configuration.dispatch`.

The **execution environment is assumed to already exist** in your registry — we
do not build one here. Reference it by name (default placeholder
`prisma-airgap-ee`).

## Apply

```bash
export CONTROLLER_HOST=https://aap.internal      # or CONTROLLER_HOSTNAME
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=...                    # or CONTROLLER_OAUTH_TOKEN
ansible-galaxy collection install -r controller/requirements.controller.yml
ansible-playbook -i controller/inventory.ini controller/configure.yml
```

Run this on a control node that can reach the Controller API — **not** inside
the air-gapped install EE. The connection collections are pinned in
`requirements.controller.yml` and are intentionally kept out of the repo-root
`requirements.yml`.

## REPLACE-ME before applying

| Placeholder | Where | Set to |
| --- | --- | --- |
| `Security Platform` | every `controller_*.yml` (`organization:`) | your AAP organization name |
| `prisma-airgap-ee` | `controller_projects.yml`, `controller_job_templates.yml` | the EE image already in your registry |
| `scm_url` | `controller_projects.yml` | this repo's internal git URL |
| `internal_backup_url`, `prisma_restore_api_user` | `controller_credentials.yml` | real non-secret values |

## Secrets

Secrets are **never committed**. They are supplied two ways:

1. **Custom credential types** (`controller_credential_types.yml`) whose
   `injectors` push the secret as an `extra_var` at job launch (masked in
   output). Credential *instances* in `controller_credentials.yml` leave the
   secret fields empty — fill them in the Controller UI after creation, or pass
   `-e @controller-secrets.vault.yml` at apply time.
2. The Controller API password / token for `configure.yml` itself comes from the
   `CONTROLLER_*` environment variables above.

Only four secrets are actually consumed by the playbooks: `prisma_backup_token`,
`internal_backup_url` (backup) and `prisma_restore_api_user`,
`prisma_restore_api_password` (DR restore). The `Prisma First-Run` credential
type stores the licence key + LDAPS bind password for safekeeping, but **no
playbook reads them** — they are entered in the Console UI during the manual
Phase 11 first-run wizard.

## Objects created

- **Organization**, **Project** (SCM → this repo, EE assigned), and **3
  inventories** (`prisma-dev`, `prisma-preproduction`, `prisma-production`) each
  sourced from `inventories/<env>/hosts.yml` so the primary/secondary site
  groups and group_vars come along on every project sync.
- **Job templates**, one per playbook: `00-baseline`, `10-podman`, `11-docker`,
  `site` (Phases 2–8), `30-ops` (Phase 10), `40-dr-drill`.
- **Two workflows** (deliberately separate — the Phase 9 installer is
  interactive and cannot run unattended):
  - `prisma-build` → runs `site` (Phases 2–8).
  - **MANUAL GATE**: operator runs `./twistlock.sh -s console` on the console
    host (Phase 9), which creates `/etc/systemd/system/twistlock.service`.
  - `prisma-ops` → runs `30-prisma-ops` (Phase 10). Launch only after the gate.

## Targeting env vs site

- **Environment** is chosen by picking the inventory at launch
  (`ask_inventory_on_launch`).
- **Site** scoping uses the launch-time **Limit** field
  (`ask_limit_on_launch`), e.g. `primary` or `secondary` — these are real
  inventory groups. Surveys are reserved for in-playbook variables; the
  functional one here is the **DR-drill survey** (`survey_dr_drill.yml`), which
  maps to the `prisma_restore_*` role inputs.

## Offline validation (no live Controller)

```bash
yamllint controller/
python -c "import yaml,glob;[yaml.safe_load(open(f)) for f in glob.glob('controller/**/*.yml',recursive=True)]"
ansible-galaxy collection install -r controller/requirements.controller.yml
ansible-playbook -i controller/inventory.ini controller/configure.yml --syntax-check
```

Keep `configure.yml` out of the GitHub Actions syntax-check — CI does not install
`infra.controller_configuration`.
