# prisma_config

## Purpose

Renders `twistlock.cfg` inside the extracted installer directory so the
operator's manual `./twistlock.sh -s console` run picks up site values
(ports, cert paths, data folder, FIPS flag). The shipped
`twistlock.cfg` is preserved as `twistlock.cfg.orig` on first run for
field-name reconciliation. Runs last in Phase 8
(`playbooks/20-prisma-stage.yml`) after `prisma_tls` and `prisma_stage`.

## Variables

This role has no `defaults/main.yml`. It consumes, from
`group_vars/prisma_console.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_extracted_dir` | `/opt/prisma-install` | Directory the staged tarball was extracted into (Palo Alto's tarball extracts flat); target for `twistlock.cfg`. |
| `prisma_https_port` | `8083` | Console UI/API port, written into `twistlock.cfg`. |
| `prisma_comm_port` | `8084` | Defender comms port. |
| `prisma_data_folder` | `/var/lib/twistlock` | Persistent data bind-mount path. |
| `prisma_fips_enabled` | `true` | FIPS flag written into the rendered config. |
| `prisma_cert_path` | `/etc/pki/prisma/console.crt` | Path to the TLS cert staged by `prisma_tls`. |
| `prisma_cert_key_path` | `/etc/pki/prisma/console.key` | Path to the TLS key. |
| `prisma_cert_ca_path` | `/etc/pki/prisma/ca-chain.crt` | Path to the CA chain. |
| `prisma_custom_cert` | `true` | Toggle for custom-cert stanza in the template. |

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** must run after `prisma_stage` (supplies
  `prisma_extracted_dir/twistlock.cfg`) and `prisma_tls` (supplies cert paths).
  Must run before the operator's manual Phase 9 (`./twistlock.sh -s console`).
- **Operator artefacts:** the shipped `twistlock.cfg` inside the Prisma tarball —
  used as the source of truth for field names when updating
  `templates/twistlock.cfg.j2`.

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
