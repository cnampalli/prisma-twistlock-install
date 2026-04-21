# prisma_storage

## Purpose

Builds the Console's persistent data volume: a single-PV volume group on
`prisma_data_device`, a 100%FREE logical volume, XFS filesystem labelled
`prisma_fs_label`, mounted at `prisma_data_folder` with
`defaults,noatime,inode64`, and SELinux-labelled `container_file_t` for
podman bind-mount. Runs last in Phase 4 (`playbooks/00-baseline.yml`),
after `rhel_baseline` and `fips_enable`.

## Variables

This role has no `defaults/main.yml`. It consumes, from
`group_vars/prisma_console.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_data_device` | `/dev/sdb` | Block device dedicated to Console data. Confirm per host — role fails if missing. |
| `prisma_vg_name` | `vg_twistlock` | LVM volume group name. |
| `prisma_lv_name` | `lv_data` | LVM logical volume name (100%FREE of VG). |
| `prisma_fs_label` | `twistlock` | XFS filesystem label. |
| `prisma_data_folder` | `/var/lib/twistlock` | Mount point and Console bind-mount target. |

## Dependencies

- **Ansible collections:**
  - `ansible.posix` — `mount` module.
  - `community.general` — `lvg`, `lvol`, `filesystem`, `sefcontext` modules.
- **Role order:** must run after `rhel_baseline` (installs `lvm2`, `xfsprogs`,
  `policycoreutils-python-utils`) and before `prisma_stage` (the installer
  tarball is staged outside the data folder but the Console needs the mount
  present before first start).
- **Operator artefacts:** a dedicated data disk attached at
  `prisma_data_device` (e.g. a VMware virtual disk). No partition table is
  created — the raw device is consumed as the PV.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  vars:
    prisma_data_device: /dev/sdb
  roles:
    - role: prisma_storage
      tags: [storage]
```

## Tags exposed

Invoked with `tags: [storage]` in `playbooks/00-baseline.yml`.

```bash
ansible-playbook playbooks/00-baseline.yml --tags storage
```
