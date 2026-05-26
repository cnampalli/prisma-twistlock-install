# AAP 2.4 Controller setup guide

How to stand up the Automation Controller (4.5.x) objects that run the Prisma
Console install — **by hand in the Controller UI**, or via the controller-as-code
in [`controller/`](../controller/).

The **execution environment (EE) is assumed to already exist** in your registry.
Everywhere below the EE is referred to as `prisma-airgap-ee` — replace it with
your real EE name.

---

## 0. Two ways to do this

| Path | When | How |
| --- | --- | --- |
| **Automated** (controller-as-code) | You can reach the Controller API from a control node and install collections | `Part A` below — one `ansible-playbook` run |
| **Manual** (Controller UI) | Air-gapped operator, no API push, or you want to click through it | `Part B` below — the bulk of this guide |

Either way the **object model is identical**; the manual steps mirror the data
in `controller/group_vars/all/`. The mapping table in §B.10 tells you which UI
object corresponds to which file.

---

## Part A — Automated apply (optional)

On a control node that can reach the Controller API:

```bash
export CONTROLLER_HOST=https://aap.internal      # or CONTROLLER_HOSTNAME
export CONTROLLER_USERNAME=admin
export CONTROLLER_PASSWORD=...                    # or CONTROLLER_OAUTH_TOKEN
ansible-galaxy collection install -r controller/requirements.controller.yml
ansible-playbook -i controller/inventory.ini controller/configure.yml
```

This creates everything in §B in one pass using the default **survey** path for
secrets — it does **not** create the optional §B.3 credential types. You still
enter the **secret** survey values at launch (§B.9) and fill the SSH key on the
`prisma-ssh` credential (§B.4). Skip to §C for the operational flow.

---

## Part B — Manual setup in the Controller UI

Do these **in order** — each step depends on the ones above it.

### Prerequisites
- **Org admin** for the target organization is enough for the default flow
  (secrets go in **surveys**, §B.9). You only need a Controller **System
  Administrator (superuser)** if you opt into the custom credential types in
  §B.3 — most install teams don't have that and should skip §B.3.
- The EE image (`prisma-airgap-ee`) already present in the Controller's registry.
- This repo reachable over git from the Controller (for the SCM project), or the
  project synced by another means.
- Decide your real names for: **Organization**, **EE**, **git URL**.

### B.1 Organization
**Access → Organizations → Add**
- **Name:** `Security Platform` *(REPLACE-ME — used by every object below)*
- **Execution Environment:** `prisma-airgap-ee` (optional default)

### B.2 Project (source of the playbooks + inventories)
**Resources → Projects → Add**
- **Name:** `prisma-twistlock-install`
- **Organization:** `Security Platform`
- **Source Control Type:** Git
- **Source Control URL:** your internal git URL for this repo *(REPLACE-ME)*
- **Source Control Branch:** `main`
- **Execution Environment:** `prisma-airgap-ee`
- **Options:** ✓ Update Revision on Launch, ✓ Clean

Save and confirm the first **Sync** succeeds (green). The project must sync
before inventories can source from it.

### B.3 Credential Types (custom) — OPTIONAL (superuser only)

> **Skip this whole section unless you are a Controller System Administrator.**
> Creating credential *types* requires superuser. The **default** path supplies
> the backup/restore secrets via **surveys** (§B.9), which only needs org-admin
> and an editable job template. Do §B.9 instead and jump past this section.
>
> Only do B.3 if you have superuser (or the platform team applies it for you)
> and prefer reusable credentials over surveys. If you create these types, then
> in §B.4 also create the matching credentials and attach them in §B.7 **instead
> of** adding the backup/restore survey questions.

**Administration → Credential Types → Add.** Create all three. For each, paste
**Input configuration** and **Injector configuration** exactly as shown.

#### Prisma Backup
Input configuration:
```yaml
fields:
  - id: prisma_backup_token
    type: string
    label: Console backup service-account token
    secret: true
  - id: internal_backup_url
    type: string
    label: Internal artefact store base URL
required:
  - prisma_backup_token
  - internal_backup_url
```
Injector configuration:
```yaml
extra_vars:
  prisma_backup_token: '{{ prisma_backup_token }}'
  internal_backup_url: '{{ internal_backup_url }}'
```

#### Prisma Restore API
Input configuration:
```yaml
fields:
  - id: prisma_restore_api_user
    type: string
    label: Console API user (restore)
  - id: prisma_restore_api_password
    type: string
    label: Console API password (restore)
    secret: true
required:
  - prisma_restore_api_user
  - prisma_restore_api_password
```
Injector configuration:
```yaml
extra_vars:
  prisma_restore_api_user: '{{ prisma_restore_api_user }}'
  prisma_restore_api_password: '{{ prisma_restore_api_password }}'
```

#### Prisma First-Run *(stored for safekeeping — no playbook consumes these)*
Holds the licence key + LDAPS bind password for the **manual Phase 11 first-run
wizard**. Create it so AAP is the single secret home; the injectors are harmless
if unused.
Input configuration:
```yaml
fields:
  - id: prisma_license_key
    type: string
    label: Prisma Cloud Compute licence key
    secret: true
  - id: prisma_ldaps_bind_password
    type: string
    label: LDAPS bind password
    secret: true
```
Injector configuration:
```yaml
extra_vars:
  prisma_license_key: '{{ prisma_license_key }}'
  prisma_ldaps_bind_password: '{{ prisma_ldaps_bind_password }}'
```

> **Why the `'{{ … }}'` syntax?** The Controller resolves the injector at job
> launch and masks secret values in job output. (In the as-code files these are
> wrapped with `!unsafe` so Ansible doesn't template them away on the push.)

### B.4 Credentials (instances)
**Resources → Credentials → Add.** Secret fields are write-only; fill them here.

The only credential the **default** path needs is the SSH one:

| Name | Credential Type | Fields to set |
| --- | --- | --- |
| `prisma-ssh` | Machine (built-in) | Username `ansible`; SSH private key for the target hosts |

All under organization `Security Platform`. The backup token + URL and the
restore API user/password are **not** credentials in the default path — they are
survey questions (§B.9).

**Only if you did the optional §B.3** also create these (otherwise skip):

| Name | Credential Type | Fields to set |
| --- | --- | --- |
| `prisma-backup-prod` | Prisma Backup | `internal_backup_url` = `https://artefacts.internal/prisma/` *(REPLACE-ME)*; `prisma_backup_token` = **secret** |
| `prisma-restore-api-prod` | Prisma Restore API | `prisma_restore_api_user` = `dr-restore` *(REPLACE-ME)*; `prisma_restore_api_password` = **secret** |
| `prisma-firstrun` *(optional)* | Prisma First-Run | licence key + LDAPS bind password (both **secret**) — for operator reference |

### B.5 Inventories
**Resources → Inventories → Add → Add inventory.** Create three:

| Name | Organization |
| --- | --- |
| `prisma-dev` | `Security Platform` |
| `prisma-preproduction` | `Security Platform` |
| `prisma-production` | `Security Platform` |

### B.6 Inventory Sources (SCM → this repo)
For **each** inventory: open it → **Sources → Add**:
- **Source:** Sourced from a Project
- **Project:** `prisma-twistlock-install`
- **Inventory file:** the matching path:
  - `prisma-dev` → `inventories/dev/hosts.yml`
  - `prisma-preproduction` → `inventories/preproduction/hosts.yml`
  - `prisma-production` → `inventories/production/hosts.yml`
- **Options:** ✓ Overwrite, ✓ Overwrite variables, ✓ Update on launch

Click **Sync** on each. After sync, the **Groups** tab should show
`prisma_all`, `prisma_console`, `prisma_scanner`, `prisma_sandbox`, `primary`,
and `secondary` — confirming the site model loaded.

### B.7 Job Templates
**Resources → Templates → Add → Add job template.** Create the six below. Shared
settings for **all** of them:
- **Job Type:** Run
- **Project:** `prisma-twistlock-install`
- **Execution Environment:** `prisma-airgap-ee`
- **Prompt on launch:** ✓ Inventory, ✓ Limit *(Limit lets you scope to `primary`
  or `secondary`)*
- **Options:** ✓ Privilege Escalation (become)
- **Credentials:** `prisma-ssh` (plus extras noted below)

| Template name | Playbook | Extra credentials | Survey |
| --- | --- | --- | --- |
| `prisma-00-baseline (Phases 2-4)` | `playbooks/00-baseline.yml` | — | — |
| `prisma-10-podman (Phase 5)` | `playbooks/10-podman.yml` | — | — |
| `prisma-11-docker (Phase 3b)` | `playbooks/11-docker.yml` | — | — |
| `prisma-site (Phases 2-8)` | `playbooks/site.yml` | — | ✓ TLS material (B.9) |
| `prisma-30-ops (Phase 10)` | `playbooks/30-prisma-ops.yml` | — | ✓ backup secrets (B.9) |
| `prisma-40-dr-drill` | `playbooks/40-dr-drill.yml` | — | ✓ DR + restore secrets (B.9) |

> The DR-drill play self-limits to `prisma_console:&secondary`, so it can only
> touch a secondary console even if you leave Limit blank.
>
> **If you took the optional §B.3 path instead:** add `prisma-backup-prod` to
> `prisma-30-ops` and `prisma-restore-api-prod` to `prisma-40-dr-drill` as extra
> credentials, and omit the secret survey questions in B.9 (keep only the DR
> behaviour questions).

### B.8 Workflow Templates
**Resources → Templates → Add → Add workflow template.** Create two. They are
**deliberately separate** because Phase 9 (the interactive `./twistlock.sh`)
cannot run unattended.

**`prisma-build`** — Phases 2–8
- Organization `Security Platform`; ✓ Prompt on launch: Inventory, Limit
- Visualizer: one node → **Job Template** `prisma-site (Phases 2-8)`

**`prisma-ops`** — Phase 10
- Organization `Security Platform`; ✓ Prompt on launch: Inventory, Limit
- Visualizer: one node → **Job Template** `prisma-30-ops (Phase 10)`

### B.9 Surveys (where the secrets go on the default path)

Surveys carry the backup/restore secrets without any custom credential type.
**Password**-type questions are masked in the UI, encrypted at rest, and injected
as `extra_vars` at launch; both templates are operator-launched, so the operator
types the secret each run (no stored default needed). Save the secret questions
with **no default**.

**B.9a — `prisma-30-ops` backup secrets.** Open `prisma-30-ops (Phase 10)` →
**Survey** → add two questions, then toggle the survey **On**:

| Prompt | Variable | Type | Default | Required |
| --- | --- | --- | --- | --- |
| Console backup service-account token | `prisma_backup_token` | **Password** | *(empty)* | ✓ |
| Internal artefact store base URL | `internal_backup_url` | Text | `https://artefacts.internal/prisma/` | ✓ |

**B.9b — `prisma-40-dr-drill` DR behaviour + restore secrets.** Open
`prisma-40-dr-drill` → **Survey** → add five questions, then toggle **On**:

| Prompt | Variable | Type | Choices | Default | Required |
| --- | --- | --- | --- | --- | --- |
| Confirm DR restore | `prisma_restore_confirmed` | Multiple Choice (single) | `false`, `true` | `false` | ✓ |
| Dry run only? | `prisma_restore_dry_run` | Multiple Choice (single) | `true`, `false` | `true` | ✓ |
| Restore source (optional) | `prisma_restore_specific_file` | Text | — | *(empty)* | — |
| Console API user (restore) | `prisma_restore_api_user` | Text | `dr-restore` | ✓ |
| Console API password (restore) | `prisma_restore_api_password` | **Password** | — | *(empty)* | ✓ |

Empty restore source = latest backup by mtime; otherwise the full path to a
specific tarball.

**B.9c — `prisma-site` TLS material.** Because the EE has no control-node
`files/` dir, the Console cert/key/CA are supplied as inline PEM. Open
`prisma-site (Phases 2-8)` → **Survey** → add three questions, then toggle **On**:

| Prompt | Variable | Type | Required |
| --- | --- | --- | --- |
| Console certificate (PEM) | `prisma_cert_content` | Textarea | ✓ |
| CA chain (PEM) | `prisma_ca_chain_content` | Textarea | ✓ |
| Console private key (PEM) | `prisma_key_content` | Textarea | ✓ |

Paste the full PEM blocks. The key uses **Textarea** (not Password) because AAP
password fields are single-line and a PEM key is multi-line — so it lands as a job
extra_var rather than masked. `prisma_tls` writes it with `no_log`, but for a
properly masked secret use the role's **Vault source** (`prisma_tls_vault_*`)
instead. For a local CLI run, pass the same vars via `-e @certs.yml`.

> **If you took the optional §B.3 path:** omit the **Password** questions above
> (`prisma_backup_token`, `prisma_restore_api_password`) and the
> `internal_backup_url` / `prisma_restore_api_user` questions — those values come
> from the attached credentials instead. Keep only the DR behaviour questions on
> `prisma-40-dr-drill`.

### B.10 Object → as-code file mapping
If you ever reconcile the UI against the repo:

| UI object | File in `controller/group_vars/all/` |
| --- | --- |
| Organization | `controller_organizations.yml` |
| Project | `controller_projects.yml` |
| Inventories | `controller_inventories.yml` |
| Inventory sources | `controller_inventory_sources.yml` |
| Credential types *(optional Path A)* | `controller_credential_types.yml` (key `…_optional`, not applied) |
| Credentials | `controller_credentials.yml` |
| Job templates | `controller_job_templates.yml` |
| Workflows | `controller_workflows.yml` |
| TLS-material survey | `survey_stage_tls.yml` |
| Backup-secrets survey | `survey_ops_backup.yml` |
| DR survey | `survey_dr_drill.yml` |

---

## Part C — Operational flow (the important bit)

The install spans an unattended part, a manual part, and an unattended part:

1. **Launch `prisma-build`** (or the `prisma-site` job template). Pick the
   inventory (`prisma-dev` / `-preproduction` / `-production`); optionally set
   **Limit** to `primary` or `secondary`. This runs Phases 2–8.
2. **Manual Phase 9 — on the console host, by hand:**
   ```bash
   cd /opt/prisma-install/prisma_cloud_compute_edition_34_01_126
   ./twistlock.sh -s console | tee /root/twistlock-install.log
   ```
   This is interactive and intentionally **not** automated; it creates
   `/etc/systemd/system/twistlock.service`. (Enter the licence key + admin/LDAPS
   details directly in the Console UI first-run wizard. The optional
   `Prisma First-Run` credential type — §B.3 — only stashes those in AAP if you
   took Path A; no playbook reads them.)
3. **Launch `prisma-ops`** (or the `prisma-30-ops` job template) against the same
   inventory. This runs Phase 10 (systemd hardening, log rotation, backup,
   monitoring) and requires the unit from step 2 to exist.
4. **DR drill (as needed):** launch `prisma-40-dr-drill`, answer the survey
   (`Confirm DR restore` = `true`, choose dry-run or real). It restores a
   **secondary** console only.

---

## Part D — Verify

- Each inventory **Sync** shows the `primary`/`secondary` groups (§B.6).
- A `prisma-site` **dry launch** prompts for inventory + limit and resolves hosts
  (use a real run only against real targets).
- DR-drill launch shows the survey and, with `prisma_restore_confirmed=false`,
  the play **aborts at the interlock** (expected).
- Secret fields on the credentials read back as `ENCRYPTED` (never plaintext).

## Notes
- **Secrets** never live in the repo. On the default path the backup/restore
  secrets are **survey** answers (§B.9 — masked in the UI, encrypted at rest);
  only the SSH key is a Controller credential (§B.4). Licence + LDAPS bind
  password are entered in the Console UI at Phase 9; no playbook reads them.
- For a **local CLI** run outside AAP, pass the same secrets via a gitignored
  file: `ansible-playbook -i inventories/<env>/hosts.yml playbooks/30-prisma-ops.yml -e @local-secrets.yml`.
- See [`controller/README.md`](../controller/README.md) for the as-code details
  and the full REPLACE-ME list.
