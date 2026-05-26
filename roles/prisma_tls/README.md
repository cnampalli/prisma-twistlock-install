# prisma_tls

## Purpose

Stages the internal-CA-issued Console TLS material (cert, key, CA chain) into
`prisma_pki_dir`, restores SELinux context, and verifies — with the **openssl
CLI** — that the cert and key public keys match and that the cert chains to the
supplied CA. Runs first in Phase 6 (`playbooks/20-prisma-stage.yml`), before
`prisma_stage` and `prisma_config`.

## Cert source (pluggable)

The material can come from three places; the source is auto-detected unless
`prisma_tls_cert_source` forces it:

| Source | When | How |
|---|---|---|
| `files` (default) | local CLI dev, `molecule/full` | `copy: src=` from the play's `files/` dir (`prisma_cert_src` etc.). |
| `content` | AAP (no control-node `files/`) | inline PEM in `prisma_cert_content`/`_key_content`/`_ca_chain_content`, supplied by a **job-template survey** (or `-e @certs.yml`). Written with `copy: content=`. |
| `vault` | when `prisma_tls_vault_addr` is set | fetched from **HashiCorp Vault** (KV v2) with the builtin `ansible.builtin.uri` module + token auth (**no `community.hashi_vault`**), then written like `content`. Inert until configured. |

Auto-detect precedence: `vault_addr` set → `vault`; elif `cert_content` set →
`content`; else `files`.

## Variables

From `roles/prisma_tls/defaults/main.yml` and `group_vars/prisma_console.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_tls_cert_source` | `auto` | `auto`/`files`/`content`/`vault`. |
| `prisma_cert_content` / `prisma_key_content` / `prisma_ca_chain_content` | `""` | Inline PEM for the `content`/`vault` sources (survey / `-e`). |
| `prisma_tls_vault_addr` / `_path` / `_token` | `""` | Vault base URL / KV v2 read path / token. Empty = Vault disabled. |
| `prisma_tls_vault_{cert,key,ca}_field` | `console.crt` / `console.key` / `ca-chain.crt` | KV field names. |
| `prisma_tls_validate` | `true` | Run the openssl match + chain checks. |
| `prisma_custom_cert` | `true` | Operator-supplied cert (vs installer self-signed). |
| `prisma_cert_src` / `prisma_key_src` / `prisma_ca_chain_src` | `console.crt` / `console.key` / `ca-chain.crt` | `files`-source filenames. |
| `prisma_pki_dir` | `/etc/pki/prisma` | Target dir (0750). |
| `prisma_cert_path` / `prisma_cert_key_path` / `prisma_cert_ca_path` | under `prisma_pki_dir` | Destinations (cert/key 0600, ca 0644). |

## Dependencies

- **Ansible collections:** none. Validation uses the `openssl` CLI; Vault fetch
  uses the builtin `ansible.builtin.uri`. (Deliberately collection-free for thin
  air-gapped EEs.)
- **Role order:** must run before `prisma_config` (writes these paths into
  `twistlock.cfg`) and `prisma_backup` (uses the CA chain as `--cacert`). Safe
  after `fips_enable`.
- **Operator artefacts:** the cert/key/CA — supplied by whichever source applies
  (`files/`, survey, or Vault). The key never lands in logs (`no_log: true`).

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
