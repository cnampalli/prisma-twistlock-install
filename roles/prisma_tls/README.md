# prisma_tls

## Purpose

Stages the internal-CA-issued Console TLS material (cert, key, CA chain)
into `prisma_pki_dir`, restores SELinux context, and asserts that the cert
and key public keys match and that the cert verifies against the supplied
CA (`openssl verify`). Runs first in Phase 6
(`playbooks/20-prisma-stage.yml`), before `prisma_stage` and `prisma_config`.

## Variables

This role has no `defaults/main.yml`. It consumes, from
`group_vars/prisma_console.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_custom_cert` | `true` | Operator-supplied cert (vs. installer self-signed). This role only runs meaningfully when true. |
| `prisma_cert_src` | `console.crt` | Source filename under the play's `files/` search path. |
| `prisma_key_src` | `console.key` | Source key filename. |
| `prisma_ca_chain_src` | `ca-chain.crt` | Source CA chain filename. |
| `prisma_pki_dir` | `/etc/pki/prisma` | Target directory for TLS material (mode 0750). |
| `prisma_cert_path` | `{{ prisma_pki_dir }}/console.crt` | Destination cert path (mode 0600). |
| `prisma_cert_key_path` | `{{ prisma_pki_dir }}/console.key` | Destination key path (mode 0600). |
| `prisma_cert_ca_path` | `{{ prisma_pki_dir }}/ca-chain.crt` | Destination CA chain path (mode 0644). |

## Dependencies

- **Ansible collections:** `community.crypto` — `x509_certificate_info`,
  `openssl_privatekey_info`.
- **Role order:** must run before `prisma_config` (writes these paths into
  `twistlock.cfg`) and `prisma_backup` (uses the CA chain as `--cacert`).
  Safe to run after `fips_enable` — FIPS-acceptable keys are expected.
- **Operator artefacts:** `console.crt`, `console.key`, `ca-chain.crt` placed
  under the control host's `files/` directory. Key files must be readable
  only by the Ansible runner; the role copies them with `no_log: true`.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  roles:
    - role: prisma_tls
      tags: [tls]
```

## Tags exposed

Invoked with `tags: [tls]` in `playbooks/20-prisma-stage.yml`.

```bash
ansible-playbook playbooks/20-prisma-stage.yml --tags tls
```
