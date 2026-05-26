# Troubleshooting — air-gapped FIPS Prisma Console install

Field notes from a real deployment: problems hit while bringing up the Prisma
Cloud Compute Console on RHEL 8.10 (FIPS, SELinux enforcing, rootful podman),
their root causes, the **host-side fix** that resolved them, and the **code
change** (now on `main`) that prevents recurrence. Ordered roughly by when they
surface in the build.

> Conventions: **Host fix** = a command you run on the box (often via the vSphere
> console). **Prevention** = behaviour now baked into the roles.

---

## 1. Baseline fails with "No inventory was parsed" — but the inventory is fine

**Symptom.** `ansible-playbook … 00-baseline.yml` aborts with:
```
ERROR! No inventory was parsed, please check your configuration and options
```
The inventory actually parses cleanly (`ansible-inventory -i inventories/dev/hosts.yml --graph` lists all hosts/groups), so the message is misleading.

**Root cause.** The real failure was a missing collection — `community.general.sefcontext` (used by `prisma_storage` for SELinux labelling). The air-gapped execution environment does **not** ship `community.general`, and operators can't add it. The generic top-level error hid this; the true cause only appeared after re-running with **maximum verbosity**:
```bash
ansible-playbook … 00-baseline.yml -vvvvv     # the missing-module error is buried here
```

**Host fix.** None — this is a code/EE issue.

**Prevention (merged).** `prisma_storage` no longer uses any collection: SELinux
labelling is done with the builtin `semanage` CLI (from `policycoreutils-python-utils`,
installed by `rhel_baseline`) instead of `community.general.sefcontext`.

**Lesson.** When an Ansible error is generic or points at the wrong thing
(inventory/config), re-run the failing task with `-vvvvv` before trusting the
headline message.

---

## 2. `/var/lib/twistlock` is not a mount → storage role won't proceed

**Symptom.** The storage step failed because `findmnt /var/lib/twistlock` found
nothing — in this environment the Console data folder is a plain directory, not a
dedicated filesystem.

**Root cause.** The role originally asserted `/var/lib/twistlock` was a dedicated
XFS mount. That assumption doesn't hold here.

**Host fix.** None needed once the role was relaxed (below). If you *do* have a
dedicated data disk, mount it at `/var/lib/twistlock` before the run.

**Prevention (merged).** `prisma_storage` was relaxed to **ensure the directory
exists + apply the SELinux label**, with no mount requirement. A dedicated volume
is recommended for capacity/IO isolation but is no longer required or asserted.

---

## 3. SSSD log-permission flood / terminal takeover (the big one)

**Symptom.**
- `[13] Permission denied` for `syslogd`/`systemd-journald` on
  `/var/log/sssd/ldap_child.log`, then `sssd.log`.
- Errors broadcast to the root terminal at regular intervals (had to keep hitting
  Ctrl-C); they also showed in `journalctl`/`ausearch -m AVC`.
- `systemctl restart sssd` itself failed: *could not open file
  /var/log/sssd/sssd.log: permission denied*.

**Root cause.** `prisma_data_folder` had been set to **`/var`** (a copy/edit
mistake, made while working around issue #2 because `/var/lib/twistlock` wasn't a
mount). With that value, `prisma_storage` ran:
```bash
semanage fcontext -a -t container_file_t '/var(/.*)?'
restorecon -R /var
```
which stamped the SELinux type **`container_file_t`** across `/var`, including
`/var/log/sssd`. Under enforcing SELinux the SSSD domain (`sssd_t`) may write
`sssd_var_log_t` but **not** `container_file_t`, so every log access was denied —
on a loop, which journald broadcast to all terminals.

**Diagnosis that pinned it:**
```bash
ls -laZ /var/log/sssd/         # showed type container_file_t (should be sssd_var_log_t)
ls -Zd /var /var/log /etc      # /var/log, /etc, /home were CORRECT -> targeted, not a system relabel
semanage fcontext -C -l        # showed the rogue '/var(/.*)?' -> container_file_t rule
```

**Host fix (confirmed working):**
```bash
semanage fcontext -d '/var(/.*)?'     # remove the rogue rule
restorecon -RFv /var                  # -F is REQUIRED: container_file_t is a "customizable
                                      # type" that plain restorecon SKIPS
systemctl restart sssd                # came back healthy
```

**Quiet the terminal while you work** (optional): `mesg n`, or set
`ForwardToWall=no` in a journald drop-in and restart `systemd-journald`. Fixing
the cause stops the flood on its own.

**Prevention (merged).** `prisma_storage` now **guards `prisma_data_folder`**: it
refuses to run unless the path is absolute, at least two levels deep, and not a
system directory (`prisma_storage_unsafe_paths` denylist). A value like `/var`
now fails fast at an assertion instead of relabelling the system.

**Lessons.**
- `container_file_t` is a *customizable type* — `restorecon` won't revert it
  without `-F`.
- A too-broad `semanage fcontext` regex + `restorecon -R` on a system path is
  destructive; always scope file-context rules to a dedicated subtree.

---

## 4. SSH fails after the FIPS reboot — "no supported authentication methods"

**Symptom.** Mid-baseline, after `fips_enable` reboots the host, Ansible (and
MobaXterm) can't reconnect:
```
Disconnected: No supported authentication methods available
(server sent: publickey, gssapi-keyex, gssapi-with-mic)
```
The vSphere **console** still logs in fine with AD credentials.

**Root cause.** The hardened host only accepts **`publickey`** and **`gssapi`**
(Kerberos) over SSH — `password` is not offered. `rhel_baseline` sets
`PasswordAuthentication no` (and the DC image is hardened similarly). The console
works because console login is PAM/SSSD, not SSH. (Advisory: **Ed25519 keys are
not FIPS-approved**, so an `id_ed25519` key would also fail post-FIPS even though
it works before — use RSA ≥2048 or ECDSA for key auth on these hosts.)

**Host fix (what was used).** `PasswordAuthentication yes` was set in
`/etc/ssh/sshd_config` and sshd reloaded, enabling MobaXterm password login.
> ⚠️ That enables password auth **globally**, which loosens the hardening
> baseline. The recommended durable form is the **scoped** exception below;
> consider reverting the global setting in favour of it.

**Prevention (merged).** `rhel_baseline` gained an **opt-in, default-off scoped
password exception**: when `rhel_baseline_password_auth_exception: true` (+
`rhel_baseline_password_auth_cidr`), it drops a `Match Address` block in
`/etc/ssh/sshd_config.d/50-prisma-admin-password.conf` that re-enables password
auth **only** from the admin network — the global `no` stays for everyone else.
(RHEL 8 `Include`s `sshd_config.d/*.conf` at the top of `sshd_config`, so the
`Match` wins for matching connections.)

**Other ways in (no password):**
- **SSH key** (FIPS-approved RSA/ECDSA — *not* Ed25519): generate locally
  (`ssh-keygen -t rsa -b 3072`), paste the `.pub` into `~/.ssh/authorized_keys`
  via the console, `restorecon -Rv ~/.ssh` (SELinux-enforcing hosts ignore a
  mislabeled `authorized_keys`), then point MobaXterm at the private key.
- **Kerberos/GSSAPI**: the host is realm-joined, so a TGT (`kinit you@REALM`, or
  Windows SSPI in MobaXterm's GSSAPI option) authenticates with your AD identity.

**Status.** Baseline now runs to completion and **FIPS activates** (confirmed).

---

## Supporting changes made in the same effort

- **Inventory** — removed placeholder `ansible_host` IPs from all
  `inventories/*/hosts.yml`; hosts connect by DNS name and the external AAP team
  owns the real connection mapping.
- **AAP secrets** — backup/restore secrets are injected via **job-template
  surveys** (password type) instead of custom credential types, because creating
  credential *types* needs Controller superuser the install team doesn't have.

---

## Quick diagnostic reference

| Need | Command |
|---|---|
| See the *real* Ansible error behind a vague one | `ansible-playbook … -vvvvv` |
| Confirm an inventory parses | `ansible-inventory -i inventories/<env>/hosts.yml --graph` |
| Check a file's SELinux label | `ls -Z <path>` / `ls -laZ <dir>` |
| List custom SELinux fcontext rules | `semanage fcontext -C -l` |
| Reset SELinux labels (incl. customizable types) | `restorecon -RFv <path>` |
| See recent SELinux denials | `ausearch -m AVC -ts recent` |
| What SSH auth methods a host offers | `ssh -v user@host` (look at "Authentications that can continue") |
| Effective sshd setting for a context | `sshd -T -C user=<u>,addr=<ip>` |
| Confirm FIPS is active | `cat /proc/sys/crypto/fips_enabled` (1) + `update-crypto-policies --show` (FIPS) |
| Is a path a mount? | `findmnt <path>` |
