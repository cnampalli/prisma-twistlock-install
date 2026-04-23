# docker_stage

## Purpose

Copies operator-supplied docker-ce RPMs (engine, CLI, containerd.io) and
their `.sha256` manifests to the target host, verifies integrity with
`sha256sum -c`, then `dnf`-installs the trio from the local paths.
Mirrors the `prisma_stage` pattern for the docker-based host classes
(`prisma_scanner`, `prisma_sandbox`). Runs in Phase 2 of the expansion
roadmap (`playbooks/11-docker.yml`), before `docker_config`.

## Variables

| Variable | Default | Description |
|---|---|---|
| `docker_ce_version` | `28.3.3` | Pinned docker engine version. |
| `docker_cli_version` | `28.3.3` | Pinned docker-ce-cli version. |
| `docker_containerd_version` | `1.7.27` | Pinned containerd.io version. |
| `docker_engine_rpm` | derived | RPM filename expected under play `files/`. |
| `docker_cli_rpm` | derived | RPM filename expected under play `files/`. |
| `docker_containerd_rpm` | derived | RPM filename expected under play `files/`. |
| `docker_stage_dir` | `/opt/docker-install` | Target staging directory. |

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** runs before `docker_config`; both roles are composed in
  `playbooks/11-docker.yml`.
- **Host mutex:** fails if `podman` is already installed. docker-ce and
  podman must not co-exist on the same host; `prisma_console` uses podman,
  `prisma_scanner` / `prisma_sandbox` use docker.
- **Operator artefacts** (dropped under the play's `files/` search path):
  - `docker-ce-<ver>-1.el8.x86_64.rpm` + `.sha256`
  - `docker-ce-cli-<ver>-1.el8.x86_64.rpm` + `.sha256`
  - `containerd.io-<ver>-1.el8.x86_64.rpm` + `.sha256`

## GPG

GPG validation is disabled in the `dnf` call because air-gapped sites
may or may not stage the docker-ce signing key. The `sha256sum -c` gate
above is the integrity check. To enable full GPG: `rpm --import` the key
out-of-band (or add a role task for it) and flip `disable_gpg_check: false`.

## Example usage

```yaml
- hosts: prisma_scanner:prisma_sandbox
  become: true
  roles:
    - role: docker_stage
      tags: [docker_stage]
```

## Tags exposed

Invoked with `tags: [docker_stage]` in `playbooks/11-docker.yml`.
