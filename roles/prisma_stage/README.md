# prisma_stage

## Purpose

Stages the Prisma Cloud Compute installer tarball and its `.sha256` manifest
to `{{ prisma_install_dir }}`, verifies integrity with `sha256sum -c`,
extracts the archive, and asserts `twistlock.sh` plus a shipped
`twistlock.cfg*` are present. Runs in Phase 7
(`playbooks/20-prisma-stage.yml`), between `prisma_tls` and `prisma_config`;
its output is what the operator's manual Phase 9 `./twistlock.sh -s console`
executes against.

## Sources

The tarball can come from one of two places, auto-detected from
`prisma_stage_nexus_path`:

| Source  | Trigger                                       | Where it reads from                                                   |
|---------|-----------------------------------------------|------------------------------------------------------------------------|
| `files` | `prisma_stage_nexus_path` is empty (default) | The play's `files/` search path on the control node. Used for local CLI dev and the molecule scenario. |
| `nexus` | `prisma_stage_nexus_path` is set             | `{{ nexus_base_url + prisma_stage_nexus_path }}/{{ prisma_tarball_name }}` + `.sha256` sidecar. Downloaded **on the controller / EE** via `delegate_to: localhost` (so the topology where the EE has Nexus reachability but the target VM does not still works), then pushed to the VM with `copy`. |

You can force a source by setting `prisma_stage_source: files` or
`prisma_stage_source: nexus` explicitly. The `nexus` source has zero
collection dependency — `get_url` is `ansible.builtin`.

**Shared Nexus config.** `nexus_base_url`, `nexus_username`, `nexus_password`,
and `nexus_validate_certs` live in `inventories/<env>/group_vars/all.yml`
and the AAP "Nexus" credential — same values across every role that fetches
from Nexus (`prisma_stage`, `docker_stage`, future). Each role only sets its
own repo path (`prisma_stage_nexus_path`, `docker_stage_nexus_path`, etc.).

## Variables

Role defaults (`defaults/main.yml`) cover all the source-mode plumbing.
Inventory `group_vars/prisma_console.yml` supplies the install-target vars:

| Variable | Default | Description |
|---|---|---|
| `prisma_version` | `34_01_126` | Must match the operator-supplied tarball filename. |
| `prisma_tarball_name` | `prisma_cloud_compute_edition_{{ prisma_version }}.tar.gz` | Used for both the files/ src and the Nexus path. |
| `prisma_install_dir` | `/opt/prisma-install` | Target directory for staging and extraction. |
| `prisma_extracted_dir` | `{{ prisma_install_dir }}` | Path the installer's contents live at post-unarchive. Palo Alto's tarball extracts flat — override per host if a future version ships a wrapper subdirectory. |
| `prisma_stage_source` | `auto` | `auto` \| `files` \| `nexus`. `auto` resolves to `nexus` when `prisma_stage_nexus_path` is set, otherwise `files`. |
| `prisma_stage_nexus_path` | `""` | Repo path appended to `{{ nexus_base_url }}`, e.g. `/repository/prisma-packages/twistlock`. |
| `prisma_stage_nexus_url` | *(computed)* | `{{ nexus_base_url + prisma_stage_nexus_path }}`. Override with a full URL only as an escape hatch. |
| `prisma_stage_nexus_username` | `{{ nexus_username }}` | Defaults to shared. Override only for a Nexus repo with different creds. |
| `prisma_stage_nexus_password` | `{{ nexus_password }}` | Defaults to shared. Secret — supply via AAP credential / survey / `-e @local-secrets.yml`. |
| `prisma_stage_nexus_validate_certs` | `{{ nexus_validate_certs }}` | Defaults to shared TLS verification setting. |
| `prisma_stage_nexus_timeout` | `600` | `get_url` timeout in seconds (tarball is GB-scale). |

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** runs after `prisma_tls` (to keep all Prisma-tarball-touching
  work downstream of TLS material) and before `prisma_config` (which renders
  `twistlock.cfg` into the extracted dir).
- **Operator artefacts (files source):**
  - `prisma_cloud_compute_edition_<ver>.tar.gz` dropped under the play's
    `files/` search path.
  - Matching `prisma_cloud_compute_edition_<ver>.tar.gz.sha256` manifest —
    the role fails if `sha256sum -c` does not print `OK`.
- **Operator artefacts (nexus source):**
  - Tarball uploaded to Nexus under
    `{{ prisma_stage_nexus_url }}/{{ prisma_tarball_name }}`.
  - SHA-256 sidecar uploaded next to it at
    `{{ prisma_stage_nexus_url }}/{{ prisma_tarball_name }}.sha256`.
  - Nexus reachable from the **controller / EE** with valid TLS (or
    `prisma_stage_nexus_validate_certs: false` in a lab). The target VM does
    **not** need to reach Nexus — the role downloads on the controller and
    pushes the artifact to the VM over SSH.

## Example usage

Local CLI, files source (default):

```yaml
- hosts: prisma_console
  become: true
  vars:
    prisma_version: "34_01_126"
  roles:
    - role: prisma_stage
      tags: [stage]
```

Nexus source, ad-hoc CLI run:

```bash
ansible-playbook -i inventories/dev/hosts.yml playbooks/20-prisma-stage.yml \
  --tags stage \
  -e nexus_base_url=https://nexus.com.au \
  -e prisma_stage_nexus_path=/repository/prisma-packages/twistlock \
  -e @local-secrets.yml   # supplies nexus_username/nexus_password
```

Nexus source, AAP: `nexus_base_url` ships in
`inventories/<env>/group_vars/all.yml`, the repo path in
`group_vars/prisma_console.yml`, and `nexus_username` / `nexus_password` come
from the shared `Nexus` AAP credential type (Path A, needs superuser) or
the password field on `survey_stage_nexus.yml` (default, no superuser). See
`docs/aap-controller-setup.md` §B.9d.

## Tags exposed

Invoked with `tags: [stage]` in `playbooks/20-prisma-stage.yml`.

```bash
ansible-playbook playbooks/20-prisma-stage.yml --tags stage
```
