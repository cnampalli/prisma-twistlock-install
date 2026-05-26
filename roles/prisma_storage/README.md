# prisma_storage

## Purpose

Ensures the Console's persistent data directory exists and labels it for podman.
It does **not** create or require a dedicated mount — whether `prisma_data_folder`
is its own filesystem is the platform/DC team's choice; the role works either
way. It:

1. **Guards** `prisma_data_folder`: refuses to run unless it is an absolute,
   dedicated path at least two levels deep and not a system directory (see
   `prisma_storage_unsafe_paths`). This stops a misconfigured value (e.g. `/var`)
   from stamping `container_file_t` across the system and breaking
   SELinux-confined services.
2. Ensures the directory exists (`0750 root:root`).
3. Registers the `container_file_t` SELinux fcontext and relabels the path so
   rootful podman can bind-mount it.

Runs last in Phase 4 (`playbooks/00-baseline.yml`), after `rhel_baseline` and
`fips_enable`.

## Variables

From `roles/prisma_storage/defaults/main.yml` and
`group_vars/prisma_console.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_data_folder` | `/var/lib/twistlock` | Console persistent-data directory. Ensured to exist and labelled `container_file_t`. Must be absolute, ≥2 levels deep, not a system dir. |
| `prisma_storage_unsafe_paths` | `[/, /var, /var/lib, /etc, /usr, …]` | Denylist `prisma_data_folder` may not be — the safety guard. |

## Dependencies

- **Ansible collections:** none. SELinux labelling uses the `semanage` CLI via
  `ansible.builtin.command` (not `community.general.sefcontext`), because the
  air-gapped EE does not ship `community.general` and operators cannot add it.
- **Role order:** must run after `rhel_baseline` (installs
  `policycoreutils-python-utils`, which provides `semanage`/`restorecon`) and
  before `prisma_stage` — the Console needs its data directory present before
  first start.
- **Operator / platform prerequisite:** none. If a dedicated data volume exists,
  mount it at `prisma_data_folder` before this runs; otherwise the role just
  creates the directory on the existing filesystem.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  roles:
    - role: prisma_storage
      tags: [storage]
```

## Tags exposed

Invoked with `tags: [storage]` in `playbooks/00-baseline.yml`.

```bash
ansible-playbook playbooks/00-baseline.yml --tags storage
```
