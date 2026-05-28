# prisma_config

## Purpose

Produces the final `twistlock.cfg` the operator's manual Phase 9
(`./twistlock.sh -s console`) sources. Runs last in Phase 8
(`playbooks/20-prisma-stage.yml`) after `prisma_tls` and `prisma_stage`.

## Strategy: seed from shipped, overlay our overrides

The role **does not** re-author `twistlock.cfg` from a Jinja template
(authoring from scratch caused a cascade of "missing field" install
failures during PR #3 — `DOCKER_TWISTLOCK_TAG` missing → podman short-name
error; `SELINUX_LABEL` missing → bad-label error; and at least nine more
fields were waiting to bite). Instead, on every run it:

1. Preserves the shipped `twistlock.cfg` (extracted by `prisma_stage`) to
   `twistlock.cfg.orig` — first run only, `force: false`.
2. Re-seeds `twistlock.cfg` from `.orig` so the baseline is deterministic.
3. Overlays just the project-specific fields listed in
   `prisma_config_overrides` via `lineinfile`.

Every field the installer expects to source (`SELINUX_LABEL=disable`,
`READ_ONLY_FS=true`, `DATA_RECOVERY_ENABLED=true`, `CLEAN_STALE_NAMESPACES`,
`RUN_CONSOLE_AS_ROOT`, `DISABLE_CONSOLE_CGROUP_LIMITS`, `CONSOLE_CN`,
`DEFENDER_CN`, etc.) inherits the shipped default. Future Prisma versions
that add new fields work without role changes.

## Variables

The role's `defaults/main.yml` ships `prisma_config_overrides` — the dict
that drives the overlay. It expands references to the inventory-supplied
console vars:

| Variable | Default | Description |
|---|---|---|
| `prisma_extracted_dir` | `/opt/prisma-install` | Directory the staged tarball was extracted into. Holds the shipped `twistlock.cfg` (overlay source) and the rendered `twistlock.cfg` (overlay target). |
| `prisma_version` | `34_01_126` | Used to compose `DOCKER_TWISTLOCK_TAG=_<ver>`. Must match the version embedded in the loaded image tags. |
| `prisma_https_port` | `8083` | Overlaid as `MANAGEMENT_PORT_HTTPS`. |
| `prisma_comm_port` | `8084` | Overlaid as `COMMUNICATION_PORT`. |
| `prisma_data_folder` | `/var/lib/twistlock` | Overlaid as `DATA_FOLDER`. |
| `prisma_fips_enabled` | `true` | Overlaid as `FIPS_ENABLED`. |
| `prisma_cert_path` | `/etc/pki/prisma/console.crt` | Overlaid as `CUSTOM_CERT_PATH`. |
| `prisma_cert_key_path` | `/etc/pki/prisma/console.key` | Overlaid as `CUSTOM_CERT_KEY_PATH`. |
| `prisma_cert_ca_path` | `/etc/pki/prisma/ca-chain.crt` | Overlaid as `CUSTOM_CERT_CA_PATH`. |
| `prisma_custom_cert` | `true` | Overlaid as `CUSTOM_CERT`. |
| `prisma_backup_dir` | `/var/lib/twistlock-backup` | Overlaid as `DATA_RECOVERY_VOLUME`. |
| `prisma_config_overrides` | *(role default)* | The mapping driving the overlay. Append in `group_vars/prisma_console.yml` to add extra fields; do not replace the role default. |

### Adding a new overlay field

In `inventories/<env>/group_vars/prisma_console.yml`:

```yaml
prisma_config_overrides: "{{ prisma_config_overrides | default({}, true) | combine({
    'READ_ONLY_FS': 'false',
    'DEFENDER_CN': inventory_hostname,
}) }}"
```

(That `default + combine` pattern preserves the role's baseline overrides
and adds yours on top.)

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** must run after `prisma_stage` (which supplies
  `prisma_extracted_dir/twistlock.cfg`) and `prisma_tls` (which supplies
  cert paths). Must run before the operator's manual Phase 9
  (`./twistlock.sh -s console`).
- **Operator artefacts:** the shipped `twistlock.cfg` inside the Prisma
  tarball — the role asserts it exists before doing anything.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  roles:
    - role: prisma_tls
    - role: prisma_stage
    - role: prisma_config
      tags: [config]
```

## Tags exposed

Invoked with `tags: [config]` in `playbooks/20-prisma-stage.yml`.

```bash
ansible-playbook playbooks/20-prisma-stage.yml --tags config
```
