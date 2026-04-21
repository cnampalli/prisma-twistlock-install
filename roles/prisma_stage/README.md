# prisma_stage

## Purpose

Copies the Prisma Cloud Compute installer tarball and its `.sha256` manifest
to `{{ prisma_install_dir }}`, verifies integrity with `sha256sum -c`,
extracts the archive, and asserts `twistlock.sh` plus a shipped
`twistlock.cfg*` are present. Runs in Phase 7
(`playbooks/20-prisma-stage.yml`), between `prisma_tls` and `prisma_config`;
its output is what the operator's manual Phase 9 `./twistlock.sh -s console`
executes against.

## Variables

This role has no `defaults/main.yml`. It consumes, from
`group_vars/prisma_console.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_version` | `34_01_126` | Must match the operator-supplied tarball filename. |
| `prisma_tarball_name` | `prisma_cloud_compute_edition_{{ prisma_version }}.tar.gz` | Source filename expected under role `files/`. |
| `prisma_install_dir` | `/opt/prisma-install` | Target directory for staging and extraction. |
| `prisma_extracted_dir` | `{{ prisma_install_dir }}/prisma_cloud_compute_edition_{{ prisma_version }}` | Extraction path asserted post-unarchive. |

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** runs after `prisma_tls` (to keep all Prisma-tarball-touching
  work downstream of TLS material) and before `prisma_config` (which renders
  `twistlock.cfg` into the extracted dir).
- **Operator artefacts:**
  - `prisma_cloud_compute_edition_<ver>.tar.gz` dropped under the play's
    `files/` search path.
  - Matching `prisma_cloud_compute_edition_<ver>.tar.gz.sha256` manifest —
    the role fails if `sha256sum -c` does not print `OK`.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  vars:
    prisma_version: "34_01_126"
  roles:
    - role: prisma_stage
      tags: [stage]
```

## Tags exposed

Invoked with `tags: [stage]` in `playbooks/20-prisma-stage.yml`.

```bash
ansible-playbook playbooks/20-prisma-stage.yml --tags stage
```
