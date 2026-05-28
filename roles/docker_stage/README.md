# docker_stage

## Purpose

Stages the pinned docker-ce RPM stack (engine, CLI, containerd.io) plus
their `.sha256` manifests to the target host, verifies integrity with
`sha256sum -c`, then `dnf`-installs the trio from the local paths.
Mirrors the `prisma_stage` pattern for the docker-based host classes
(`prisma_scanner`, `prisma_sandbox`). Runs in Phase 2 of the expansion
roadmap (`playbooks/11-docker.yml`), before `docker_config`.

## Sources

The RPMs can come from one of two places, auto-detected from
`docker_stage_nexus_path`:

| Source  | Trigger                                       | Where it reads from                                                   |
|---------|-----------------------------------------------|------------------------------------------------------------------------|
| `files` | `docker_stage_nexus_path` is empty (default) | The play's `files/` search path on the control node. Used for local CLI dev and the molecule scenario. |
| `nexus` | `docker_stage_nexus_path` is set             | `{{ nexus_base_url + docker_stage_nexus_path }}/{{ rpm_filename }}` (× 6 — 3 RPMs + 3 sidecars). Downloaded **on the controller / EE** via `delegate_to: localhost`, then pushed to the VM with `copy`. Works in the topology where the EE has Nexus reachability but the scanner/sandbox VMs do not. |

Force a source with `docker_stage_source: files` or `docker_stage_source: nexus`.

**Shared Nexus config.** Same credentials and base URL as `prisma_stage`:
`nexus_base_url`, `nexus_username`, `nexus_password`, `nexus_validate_certs`
live in `inventories/<env>/group_vars/all.yml` + the AAP "Nexus" credential.

## Variables

| Variable | Default | Description |
|---|---|---|
| `docker_ce_version` | `28.3.3` | Pinned docker engine version. |
| `docker_cli_version` | `28.3.3` | Pinned docker-ce-cli version. |
| `docker_containerd_version` | `1.7.27` | Pinned containerd.io version. |
| `docker_engine_rpm` | derived | RPM filename used by both files/ and nexus sources. |
| `docker_cli_rpm` | derived | RPM filename. |
| `docker_containerd_rpm` | derived | RPM filename. |
| `docker_stage_dir` | `/opt/docker-install` | Target staging directory. |
| `docker_stage_source` | `auto` | `auto` \| `files` \| `nexus`. |
| `docker_stage_nexus_path` | `""` | Repo path appended to `{{ nexus_base_url }}`, e.g. `/repository/docker-packages/rhel8`. |
| `docker_stage_nexus_url` | *(computed)* | `{{ nexus_base_url + docker_stage_nexus_path }}`. Escape hatch only. |
| `docker_stage_nexus_username` | `{{ nexus_username }}` | Defaults to shared. |
| `docker_stage_nexus_password` | `{{ nexus_password }}` | Defaults to shared. Secret — `no_log` on fetch tasks. |
| `docker_stage_nexus_validate_certs` | `{{ nexus_validate_certs }}` | Defaults to shared. |
| `docker_stage_nexus_timeout` | `300` | `get_url` timeout per RPM. |

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** runs before `docker_config`; both roles are composed in
  `playbooks/11-docker.yml`.
- **Host mutex:** fails if `podman` is already installed. docker-ce and
  podman must not co-exist on the same host; `prisma_console` uses podman,
  `prisma_scanner` / `prisma_sandbox` use docker.
- **Operator artefacts (files source):**
  - `docker-ce-<ver>-1.el8.x86_64.rpm` + `.sha256`
  - `docker-ce-cli-<ver>-1.el8.x86_64.rpm` + `.sha256`
  - `containerd.io-<ver>-1.el8.x86_64.rpm` + `.sha256`
- **Operator artefacts (nexus source):** the same 6 files uploaded to Nexus
  under `{{ nexus_base_url + docker_stage_nexus_path }}/`. Reachable from
  the controller / EE; target VM does not need direct Nexus access.

## GPG

GPG validation is disabled in the `dnf` call because air-gapped sites
may or may not stage the docker-ce signing key. The `sha256sum -c` gate
is the integrity check. To enable full GPG: `rpm --import` the key
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
