# podman_config

## Purpose

Configures rootful podman on the Console host for air-gapped, FIPS-mode
operation: renders `/etc/containers/{registries,storage,containers}.conf`,
installs the internal registry CA under `/etc/containers/certs.d/`, masks the
podman API socket, and smoke-tests `podman info` for overlay/seccomp/SELinux.
Runs alone in Phase 5 (`playbooks/10-podman.yml`) between baseline/FIPS/storage
and the Prisma staging phase — see
`/Users/chait/vachan-homelab/projects/prisma-twistlock-install/playbooks/site.yml`.

## Variables

From `roles/podman_config/defaults/main.yml`:

| Variable | Default | Description |
|---|---|---|
| `podman_internal_registry` | `registry.internal` | FQDN of the air-gapped registry that hosts Prisma/Defender images. |
| `podman_blocked_registries` | `[docker.io, quay.io, registry.redhat.io, registry.access.redhat.com]` | Registries explicitly blocked; public egress indicates a misconfigured pull. |
| `podman_log_size_max` | `104857600` | Per-container log cap in bytes (100 MB). |
| `podman_pids_limit` | `4096` | Per-container PID ceiling. |
| `podman_nofile` | `"65535:65535"` | ulimit nofile soft:hard, per Prisma docs. |
| `podman_nproc` | `"4096:4096"` | ulimit nproc soft:hard. |
| `podman_cgroup_manager` | `systemd` | Cgroup manager; required for RHEL 9 cgroup v2. |
| `podman_runtime` | `crun` | FIPS-validated OCI runtime on RHEL 9. |
| `podman_registry_ca_cert` | `""` | Absolute path on the control host to a PEM CA cert for `podman_internal_registry`. Empty means rely on the system trust store. |

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** must run after `rhel_baseline` (installs `podman`) and
  after `fips_enable` (so `crun` is selected under the FIPS kernel). Must run
  before `prisma_stage` — the installer will later pull images via podman.
- **Operator artefacts:** optional PEM CA cert at `podman_registry_ca_cert` if
  the internal registry uses a private CA.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  roles:
    - role: podman_config
      vars:
        podman_internal_registry: registry.corp.example
        podman_registry_ca_cert: files/registry-ca.crt
      tags: [podman]
```

## Tags exposed

Invoked with `tags: [podman]` in `playbooks/10-podman.yml`.

```bash
ansible-playbook playbooks/10-podman.yml --tags podman
```
