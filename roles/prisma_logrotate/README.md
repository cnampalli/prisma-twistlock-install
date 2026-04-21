# prisma_logrotate

## Purpose

Installs `/etc/logrotate.d/prisma-console` from template and validates it
with `logrotate -d`. Keeps the Console application log volume bounded
without losing debug context between rsyslog ships. Runs in Phase 10
(`playbooks/30-prisma-ops.yml`), second role after `prisma_systemd`.

## Variables

This role has no `defaults/main.yml`. The template
`prisma-console.logrotate.j2` reads site-wide values from
`group_vars/prisma_console.yml`:

| Variable | Default | Description |
|---|---|---|
| `prisma_data_folder` | `/var/lib/twistlock` | Console data folder; log paths under this directory are the rotation targets. |

No role-local variables are defined.

## Dependencies

- **Ansible collections:** `ansible.builtin` only.
- **Role order:** must run after `prisma_systemd` (Console unit is up and
  writing logs) and after `rhel_baseline` (installs the `logrotate` package).
- **Operator artefacts:** none.

## Example usage

```yaml
- hosts: prisma_console
  become: true
  roles:
    - role: prisma_logrotate
      tags: [logrotate]
```

No variables need to be overridden.

## Tags exposed

Invoked with `tags: [logrotate]` in `playbooks/30-prisma-ops.yml`.

```bash
ansible-playbook playbooks/30-prisma-ops.yml --tags logrotate
```
