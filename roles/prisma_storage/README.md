# prisma_storage

## Purpose

Verifies the Console's persistent data volume is in place and labels it for
podman. The platform/DC team provisions and mounts the data volume at
`prisma_data_folder` **before VM handover**; this role does not create the
volume group, logical volume, filesystem, or mount. It:

1. Asserts `prisma_data_folder` is a dedicated XFS mount (not NFS) via `findmnt`.
2. Registers the `container_file_t` SELinux fcontext and relabels the path so
   rootful podman can bind-mount it.

Runs last in Phase 4 (`playbooks/00-baseline.yml`), after `rhel_baseline` and
`fips_enable`.

## Variables

This role has no `defaults/main.yml`. It consumes, from
`group_vars/prisma_console.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_data_folder` | `/var/lib/twistlock` | Mount point the DC team provisions; the role asserts it is a dedicated XFS mount and labels it `container_file_t`. |

## Dependencies

- **Ansible collections:**
  - `community.general` — `sefcontext` module.
- **Role order:** must run after `rhel_baseline` (installs
  `policycoreutils-python-utils` for `restorecon`/`sefcontext`) and before
  `prisma_stage` — the Console needs the data mount present before first start.
- **Operator / platform prerequisite:** the data volume must already be mounted
  at `prisma_data_folder` as XFS (not NFS) when the role runs. The role fails
  fast with remediation guidance if it is missing.

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
