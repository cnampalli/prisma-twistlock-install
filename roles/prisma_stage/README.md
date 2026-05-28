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
`prisma_stage_nexus_url`:

| Source  | Trigger                                  | Where it reads from                                                   |
|---------|------------------------------------------|------------------------------------------------------------------------|
| `files` | `prisma_stage_nexus_url` is empty (default) | The play's `files/` search path on the control node. Used for local CLI dev and the molecule scenario. |
| `nexus` | `prisma_stage_nexus_url` is set          | `{{ prisma_stage_nexus_url }}/{{ prisma_tarball_name }}` + `.sha256` sidecar, via `ansible.builtin.get_url`. Used in AAP execution environments, which have no control-node `files/` dir. |

You can force a source by setting `prisma_stage_source: files` or
`prisma_stage_source: nexus` explicitly. The `nexus` source has zero
collection dependency — `get_url` is `ansible.builtin`.

## Variables

Role defaults (`defaults/main.yml`) cover all the source-mode plumbing.
Inventory `group_vars/prisma_console.yml` supplies the install-target vars:

| Variable | Default | Description |
|---|---|---|
| `prisma_version` | `34_01_126` | Must match the operator-supplied tarball filename. |
| `prisma_tarball_name` | `prisma_cloud_compute_edition_{{ prisma_version }}.tar.gz` | Used for both the files/ src and the Nexus path. |
| `prisma_install_dir` | `/opt/prisma-install` | Target directory for staging and extraction. |
| `prisma_extracted_dir` | `{{ prisma_install_dir }}` | Path the installer's contents live at post-unarchive. Palo Alto's tarball extracts flat — override per host if a future version ships a wrapper subdirectory. |
| `prisma_stage_source` | `auto` | `auto` \| `files` \| `nexus`. `auto` resolves to `nexus` when `prisma_stage_nexus_url` is set, otherwise `files`. |
| `prisma_stage_nexus_url` | `""` | Base URL of the Nexus dir, e.g. `https://nexus.com.au/repository/prisma-packages/twistlock`. |
| `prisma_stage_nexus_username` | `""` | Optional basic-auth user. Blank = anonymous read. |
| `prisma_stage_nexus_password` | `""` | Optional basic-auth password. Secret — supply via AAP credential / survey / `-e @local-secrets.yml`. |
| `prisma_stage_nexus_validate_certs` | `true` | TLS verification for the Nexus host. |
| `prisma_stage_nexus_timeout` | `600` | `get_url` timeout in seconds. |

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
  - Nexus reachable from the EE / control node with valid TLS (or
    `prisma_stage_nexus_validate_certs: false` in a lab).

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
  -e prisma_stage_nexus_url=https://nexus.com.au/repository/prisma-packages/twistlock \
  -e @local-secrets.yml   # supplies prisma_stage_nexus_username/password
```

Nexus source, AAP: the URL ships in `inventories/<env>/group_vars/prisma_console.yml`,
and the username/password come from the `Prisma Nexus` credential type (Path A,
needs superuser) or the password field on `survey_stage_nexus.yml` (default,
no superuser). See `docs/aap-controller-setup.md` §B.9d.

## Tags exposed

Invoked with `tags: [stage]` in `playbooks/20-prisma-stage.yml`.

```bash
ansible-playbook playbooks/20-prisma-stage.yml --tags stage
```
