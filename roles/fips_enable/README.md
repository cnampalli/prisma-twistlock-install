# fips_enable

## Purpose

Enables RHEL FIPS 140-2 mode via `fips-mode-setup --enable`, reboots the host
to activate the FIPS kernel, then asserts `/proc/sys/crypto/fips_enabled` is
`1` and the system crypto policy is `FIPS`. Runs in Phase 2 (baseline), second
role in `playbooks/00-baseline.yml` — after `rhel_baseline` installs packages
and before `prisma_storage` builds the data volume. See
`/Users/chait/vachan-homelab/projects/prisma-twistlock-install/playbooks/site.yml`.

## Variables

This role has no `defaults/main.yml`. It consumes one variable from
`group_vars/prisma_console.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_fips_enabled` | `true` | Gate for the baseline playbook. When `false`, include this role conditionally; `prisma_validate` will fail because FIPS is a hard Prisma Console requirement. |

No role-local variables are defined.

## Dependencies

- **Ansible collections:** none beyond `ansible.builtin`.
- **Role order:** must run after `rhel_baseline` (which ensures
  `policycoreutils` and `crypto-policies` are present) and before any role
  that materialises TLS keys — FIPS changes which key types are accepted.
- **Operator artefacts:** none. RHEL ships `fips-mode-setup` in `crypto-policies-scripts`.
- **Runtime side-effect:** issues a reboot via `ansible.builtin.reboot` with a
  600s timeout. The managed host must be reachable after reboot.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  roles:
    - role: fips_enable
      tags: [fips]
```

No variables are required — the role is self-contained.

## Tags exposed

Invoked with `tags: [fips]` in `playbooks/00-baseline.yml`.

```bash
ansible-playbook playbooks/00-baseline.yml --tags fips
```
