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
`rhel_baseline_password_auth_cidrs` — a **list** of CIDRs joined into sshd's
`Match Address` line), it drops a `Match Address` block in
`/etc/ssh/sshd_config.d/50-prisma-admin-password.conf` that re-enables password
auth **only** from the listed networks — the global `no` stays for everyone else.
(RHEL 8 `Include`s `sshd_config.d/*.conf` at the top of `sshd_config`, so the
`Match` wins for matching connections.)

**Update (merged).** Originally `rhel_baseline_password_auth_cidr` was a single
string with one CIDR. That bit operators whose Ansible / MobaXterm source IP
wasn't in the one admin range (e.g. AAP EE on a different subnet, jump host
network, lab subnet) — they'd hit this exact issue *again* after the FIPS
reboot and need a console login to unstick the playbook. The variable is now
the plural `_cidrs` (list); list **every** legitimate source network in
`inventories/<env>/group_vars/all.yml`. The role asserts the list is non-empty
when the exception is enabled. If you set the singular form, the role will
fail-fast with a clear migration message instead of silently rendering the
wrong config.

**Other ways in (no password):**
- **SSH key** (FIPS-approved RSA/ECDSA — *not* Ed25519): generate locally
  (`ssh-keygen -t rsa -b 3072`), paste the `.pub` into `~/.ssh/authorized_keys`
  via the console, `restorecon -Rv ~/.ssh` (SELinux-enforcing hosts ignore a
  mislabeled `authorized_keys`), then point MobaXterm at the private key.
- **Kerberos/GSSAPI**: the host is realm-joined, so a TGT (`kinit you@REALM`, or
  Windows SSPI in MobaXterm's GSSAPI option) authenticates with your AD identity.

**Status.** Baseline now runs to completion and **FIPS activates** (confirmed).

---

## 5. Tarball extracts flat — `prisma_extracted_dir/twistlock.sh` not found

**Symptom.** `prisma_stage` succeeds the unarchive step but the very next
assertion or downstream task fails on a path under
`/opt/prisma-install/prisma_cloud_compute_edition_<ver>/`. The operator's
manual `cd $prisma_extracted_dir` also reports "No such file or directory",
and `prisma_config` / `prisma_restore` later point at non-existent paths.

**Root cause.** The Prisma Cloud Compute tarball extracts **flat** into the
target dir — `twistlock.sh`, `twistlock.cfg*`, `linux/twistcli`, and
`images/` all land directly under `/opt/prisma-install/`. The role originally
defaulted `prisma_extracted_dir` to a `prisma_cloud_compute_edition_<ver>/`
wrapper subdirectory that the tarball does **not** create.

**Host fix.** None — once the default is corrected the role runs to
completion. If you ran the role before the fix landed, no cleanup is needed;
the flat-extracted files are already in the right place.

**Prevention (merged, `61114c6`).** `prisma_extracted_dir` default in all
three inventories' `prisma_console.yml` flipped from
`{{ prisma_install_dir }}/prisma_cloud_compute_edition_{{ prisma_version }}`
to `{{ prisma_install_dir }}`. A comment notes the assumption — operators on
a future Prisma version that *does* ship with a wrapper dir can override per
host. The three manual-install guides
(`docs/manual-install/{console,image-scanner,sandbox}.md`) got the same
correction in `687a0c2`.

---

## 6. `twistlock.sh` dies on `short-name "twistlock/private:console" did not resolve`

**Symptom.** The manual Phase 9 `./twistlock.sh -s console` step prints:
```
Loaded image: localhost/twistlock/private:console_34_01_126
Error: you must provide at least one name or id
Error: short-name "twistlock/private:console" did not resolve to an alias and
       no unqualified-search registries are defined in "/etc/containers/registries.conf"
Failed to start twistlock data container
```
Note the tag in the error: **`console` with no `_34_01_126` suffix**.

**Root cause.** Two compounding bugs:

1. `twistlock.sh` composes image names as
   `twistlock/private:console${DOCKER_TWISTLOCK_TAG}` (around line 1593 of
   the shipped script). With `DOCKER_TWISTLOCK_TAG` undefined the suffix
   collapses to empty, hence the bare `:console` in the error.
2. Even with the tag, `twistlock/private:console_<ver>` is an unqualified
   short name; the hardened `registries.conf` has
   `unqualified-search-registries = []` and no `[aliases]` table, so podman
   cannot resolve it to the loaded
   `localhost/twistlock/private:console_<ver>` image.

**Host fix.** Re-render the two affected files and re-run the installer:
```bash
ansible-playbook -i inventories/<env>/hosts.yml playbooks/site.yml --tags podman,config
cd /opt/prisma-install && ./twistlock.sh -s console
```

**Prevention (merged, `5df6a61`).** Two coordinated edits:

- `roles/prisma_config/templates/twistlock.cfg.j2` adds
  `DOCKER_TWISTLOCK_TAG=_{{ prisma_version }}` so the tag is sourced correctly.
- `roles/podman_config/templates/registries.conf.j2` adds a narrow
  `[aliases]` table mapping `twistlock/private` to
  `localhost/twistlock/private` — per-image specific, does **not** loosen
  short-name resolution for any other repo. (Setting
  `unqualified-search-registries = ["localhost"]` instead would have
  weakened the hardening for everything; the alias is the narrower fix.)

---

## 7. `twistlock.sh` dies on `Bad label option ""` — and the wider missing-cfg-field class

**Symptom.** After the short-name fix (#6 above), `./twistlock.sh -s console`
gets further but dies with:
```
Error: Creating container storage: Bad label option "", valid options 'disable' or
       'user, role, level, type, filetype' followed by ':' and a value.
Failed to start Twistlock data container
```
Or with slightly different wording every time the installer reaches a new
step that needs a cfg field our render didn't include.

**Root cause.** The original `prisma_config/templates/twistlock.cfg.j2`
re-authored `twistlock.cfg` from scratch with 9 fields. The shipped
`twistlock.cfg.original` defines ~25 fields, and `twistlock.sh` sources them
all. Every time the installer touched a field we hadn't rendered, the value
was empty and the install died:

- `DOCKER_TWISTLOCK_TAG` missing → short-name resolution failure (#6).
- `SELINUX_LABEL=disable` missing → `--security-opt label=` (empty value) →
  "Bad label option" above.
- `READ_ONLY_FS`, `DATA_RECOVERY_ENABLED`, `CLEAN_STALE_NAMESPACES`,
  `DISABLE_CONSOLE_CGROUP_LIMITS`, `RUN_CONSOLE_AS_ROOT`, `DEFENDER_CN`,
  `CONSOLE_CN`, etc. were all queued up to bite next.

**Host fix.** Re-render the cfg and re-run the installer:
```bash
ansible-playbook -i inventories/<env>/hosts.yml playbooks/site.yml --tags config
cd /opt/prisma-install && ./twistlock.sh -s console
```

**Prevention (merged, `60ee440`).** `prisma_config` no longer authors the cfg.
The role now:

1. Asserts `prisma_stage` extracted the shipped `twistlock.cfg`.
2. Preserves it as `twistlock.cfg.orig` on first run (source of truth for
   the overlay; `force: false`).
3. Re-seeds `twistlock.cfg` from `.orig` every run (deterministic baseline).
4. Overlays just the 10 fields listed in `prisma_config_overrides` via
   `lineinfile`.

Every other field inherits its shipped default — including
`SELINUX_LABEL=disable` and the nine that hadn't bitten yet. Operators
extend `prisma_config_overrides` in `group_vars/prisma_console.yml` to add
more overrides without patching the role.

**Lesson.** Two failures of the same shape ("missing cfg field" → install
dies) is a *Phase-4.5 architecture signal* per the systematic-debugging
discipline, not "fix the next field individually." A diff-based render
(`shipped + overlay`) self-corrects against new fields in future Prisma
versions.

---

## 8. EE has Nexus reachability but the target VM does not — `get_url` hangs/refuses

**Symptom.** `prisma_stage` running from an AAP execution environment with
the Nexus source path fails the tarball download with a connection /
timeout / "name or service not known" error, even though `curl -I <nexus>`
from the EE itself works fine. The failure traceback points at a task
running **on the target host**.

**Root cause.** By default `ansible.builtin.get_url` (and friends) run **on
the target host**, not the controller. In hardened topologies where the AAP
execution environment has Nexus reachability but the target VM sits in a
no-egress subnet, the download attempt happens from the wrong side of the
network boundary.

**Host fix.** None — the fix is in the role, not on the host. (Opening
VM → Nexus connectivity to "work around" it usually breaks the air-gap
posture the topology is enforcing.)

**Prevention (merged, `beb0655`).** `prisma_stage` Nexus block refactored
into an **EE-relay**:

1. `ansible.builtin.tempfile` creates a controller-side staging dir
   (`delegate_to: localhost`).
2. `ansible.builtin.get_url` downloads the tarball + `.sha256` onto the EE
   (`delegate_to: localhost`, `no_log: true`).
3. `ansible.builtin.copy` pushes both artifacts from controller → VM over
   SSH (default delegation — runs on the VM, reading the source from the
   controller).
4. `always: file state=absent` cleans up the controller-side staging dir
   even on failure.

The same pattern lands on `docker_stage` in PR #4 (Nexus consolidation).
**Operator implication:** Nexus must be reachable from the controller / EE,
**not** from the target VM. Verify with `curl -I <nexus-url>` from **inside
the execution environment**, not from the install target.

---

## 9. Scanner/sandbox chrony starts but never syncs — silent NTP miss

**Symptom.** chronyd is `active (running)` on a scanner or sandbox host but
`chronyc sources` shows zero peers (output: `MS Name/IP address` header with
no rows, or `Number of sources = 0`). `timedatectl` reports
`System clock synchronized: no`. Console-to-Defender time drift creeps past
the 30-second contract Prisma requires; alerting may flag a Defender as
unreachable purely because of clock skew. **No error appears in the playbook
run — the install reports green.**

**Root cause.** `rhel_baseline` runs on `hosts: prisma_all` (console +
scanner + sandbox) and renders `/etc/chrony.conf` from a template that
unconditionally loops over `ntp_servers`. The variable was historically
defined only in `inventories/<env>/group_vars/prisma_console.yml`, so the
Console host inherited a working list while scanner/sandbox got an empty
Jinja loop → conf with no `server` lines → chronyd starts cleanly with
nothing to sync against.

**Host fix.** Re-render `/etc/chrony.conf` and restart chronyd:
```bash
ansible-playbook -i inventories/<env>/hosts.yml playbooks/00-baseline.yml \
  --tags baseline,ntp --limit prisma_scanner:prisma_sandbox
# Then on each host, confirm peers show up:
chronyc sources
timedatectl
```

**Prevention (merged).** Two coordinated edits:

- `ntp_servers` relocated from `prisma_console.yml` to `prisma_all.yml`
  across all three envs. The `prisma_all` scope is the canonical home for
  rhel_baseline knobs shared by every host class — comment in the file
  itself invites this use case.
- `rhel_baseline` gained an `ansible.builtin.assert` immediately before the
  chrony.conf template task, with `that: ntp_servers | length > 0` and a
  clear `fail_msg` pointing the operator at `prisma_all.yml`. Future runs
  with an empty/undefined list fail at the assert, not silently downstream.

**Lesson.** A role that runs on multiple host classes and depends on a
class-specific inventory var is a silent-failure waiting to happen. When
adding any such role-required var, put it at the broadest applicable scope
(`all.yml` or `prisma_all.yml`) and add a fail-fast assert in the role —
never trust "the right inventory file will have it."

---

## 10. `docker_config` asserts `'systemd' in docker_info.stdout` but `docker info` reports `cgroupfs`

**Symptom.** On a fresh scanner/sandbox install, `docker_config` fails at the
"Assert docker runtime properties" task with:
```
fail_msg: docker info did not report expected config — check /etc/docker/daemon.json
```
`cat /etc/docker/daemon.json` shows `"exec-opts": ["native.cgroupdriver=systemd"]`
correctly. The mismatch is between what the file says and what the running
daemon reports:
```bash
$ sudo docker info --format '{{.Driver}} {{.CgroupDriver}} {{.SecurityOptions}}'
overlay2 cgroupfs [name=seccomp,profile=builtin]
                ^^^^^^^^^^
                # daemon.json says systemd, daemon is on cgroupfs
```

**Root cause.** Classic Ansible handler-ordering pitfall. The role's flow:

1. Render `daemon.json` → `notify: Restart docker` (handler queued, doesn't fire).
2. Render `override.conf` → `notify: Restart docker` (queued, deduplicated).
3. `systemd: state=started, daemon_reload=true` — `daemon_reload: true` only
   reloads **systemd's view of unit files**, NOT the docker daemon's config.
   If docker was already running (e.g. dnf's `docker-ce` post-install hook
   started it with package defaults), this task is a no-op.
4. Smoke test queries `docker info` against the **stale running daemon**
   still on its initial cgroupfs/etc. defaults.
5. Handler doesn't fire until end-of-play — too late.

The `daemon.json` value is correct; the daemon never read it. journalctl
shows no warnings because docker never *tried* to apply the new config.

**Host fix.** Restart docker manually, then re-run:
```bash
sudo systemctl restart docker
sudo docker info --format '{{.Driver}} {{.CgroupDriver}} {{.SecurityOptions}}'
# expect: overlay2 systemd [name=seccomp,profile=builtin]
ansible-playbook -i inventories/<env>/hosts.yml playbooks/11-docker.yml --tags docker_config
```

**Prevention (merged).** `roles/docker_config/tasks/main.yml` gained a
`meta: flush_handlers` task between the systemd start task and the smoke
test. This forces the queued `Restart docker` handler to fire **before**
`docker info` runs, so the smoke test sees the post-restart config.

**Lesson.** Any role that (a) renders a config file with `notify: Restart X`
and (b) immediately asserts the live state of service X is exposed to this
pitfall. The fix is always `meta: flush_handlers` between the renders and
the asserts. Worth a code-search for the pattern across other roles
(`prisma_systemd`, `podman_config`, etc.) as a follow-up.

---

## 11. `dockerd` crashes on startup: `unable to configure ... directives don't match any configuration option: _comment`

**Symptom.** `systemctl restart docker` exits non-zero, journalctl shows:
```
dockerd[16237]: unable to configure the Docker daemon with file
/etc/docker/daemon.json: the following directives don't match any
configuration option: _comment
docker.service: Main process exited, code=exited, status=1/FAILURE
docker.service: Failed with result 'exit-code'.
docker.service: Start request repeated too quickly.
```
The daemon never finishes initializing. `docker info`, `docker ps`, etc.
all fail because there's no daemon to talk to. This surfaces the *first*
time docker actually tries to **read** the role-rendered daemon.json —
either after `meta: flush_handlers` (post-fix #10), or after a manual
`systemctl restart docker`, or on a host where the role-rendered config
landed before docker was first started.

**Root cause.** Docker's daemon.json parser is **strict**: it errors on
any unrecognized key, even ones prefixed with `_` (the Linux convention
for "ignore me, I'm a comment"). The `roles/docker_config/templates/daemon.json.j2`
template carried a leading `"_comment": "Managed by Ansible …"` line as
a poor-man's comment (JSON has no native comments). Docker rejects it and
exits 1.

This bug was hidden behind issue #10 (handler-ordering): docker was never
restarted, so it never read the file. The instant the daemon actually
loaded daemon.json — whether by the new `meta: flush_handlers` step or
by an operator manually restarting — it crash-looped. Two-bug latency
chain: the fix for #10 surfaces #11.

**Host fix.** Edit `/etc/docker/daemon.json`, delete the `"_comment": "..."`
line (and the trailing comma if your file places it before another key),
then:
```bash
sudo systemctl start docker
sudo docker info --format '{{.CgroupDriver}}'   # expect: systemd
```

**Prevention (merged).** `_comment` removed from `daemon.json.j2`. The
file's provenance is captured by:
- The role's git history (`git log roles/docker_config/templates/daemon.json.j2`).
- The file's mtime + 0644 root:root ownership.
- The fact that `docker_config` is the only thing in the codebase that
  writes `/etc/docker/daemon.json`.

If you genuinely need an "Ansible managed" marker in the rendered file,
write it to a sidecar (e.g. `/etc/docker/.daemon.json.managed`) instead
of into the JSON itself. Don't add unknown keys to daemon.json.

**Lesson.** When emitting structured config files (JSON, TOML, etc.),
verify the parser's tolerance for unknown keys before adding any
"marker" fields. JSON has no comment syntax; convention is to put
provenance metadata in a sidecar file or in surrounding documentation,
not in the data file itself. Audit other templates that render strict
formats (`registries.conf` — TOML allows comments, `chrony.conf` — `#`
comments work, `daemon.json` — **no comments**) for the same trap.

---

## 12. `dockerd` crashes on startup: `unknown log opt 'max-size' for journald log driver`

**Symptom.** After issues #10 and #11 are resolved (handlers flush, `_comment`
removed), docker still fails to start with:
```
failed to start daemon: failed to set log opts: failed to set log opts:
unknown log opt 'max-size' for journald log driver
docker.service: Main process exited, code=exited, status=1/FAILURE
```
Also visible in the journal (not a failure, just noise — disregard):
```
level=info msg="CDI directory does not exist, skipping: ... dir=/etc/cdi"
level=info msg="CDI directory does not exist, skipping: ... dir=/var/run/cdi"
```
The CDI lines are informational (optional GPU/device passthrough). The
`unknown log opt` line is fatal.

**Root cause.** `max-size` / `max-file` are **`json-file`** log driver options
— they are **NOT valid** for the **`journald`** log driver. The
`daemon.json.j2` template was emitting them unconditionally alongside
`"log-driver": "journald"`. Journald handles its own rotation via
`/etc/systemd/journald.conf` (the `SystemMaxUse=` cap, which `rhel_baseline`
already manages via `rhel_baseline_journald_max_use: 2G`); docker doesn't
expose those knobs.

**Host fix.** Edit `/etc/docker/daemon.json`, delete the entire `"log-opts": { … }`
block (and its trailing comma if needed), then:
```bash
sudo systemctl start docker
sudo journalctl -u docker --since "2 min ago" --no-pager | tail -5
# expect: docker.service: Started successfully ...
```

**Prevention (merged).** `daemon.json.j2` now wraps the `log-opts` block in
`{% if docker_log_driver == 'json-file' %} … {% endif %}`. With the default
`journald` driver the block disappears entirely; if an operator overrides
`docker_log_driver: json-file`, log-opts come back. Backward-compatible at
the var layer — `docker_log_size_max` / `docker_log_max_file` defaults stay
defined for the json-file case.

**Lesson — three bugs in one role, one architectural cause.** This is the
**third** docker_config bug in the same install: #10 (handlers not flushed),
#11 (`_comment` rejected), #12 (log-opts/driver mismatch). All three share
a root: the role's smoke test only runs `docker info` **after** the daemon
has been restarted with our config — so any config that docker REJECTS at
startup turns into "daemon crash loop with no specific assertion." A role
that emits strict-parser config files (JSON, TOML, INI) should ideally
validate the rendered file BEFORE killing the running service — e.g.
`dockerd --validate /etc/docker/daemon.json` or equivalent — so a bad
template fails the playbook with a clear "this option is invalid" message
rather than a generic `status=1/FAILURE`. Worth a follow-up across other
strict-format renders (`registries.conf` is TOML — tolerates comments;
`chrony.conf` accepts `#`; **`daemon.json` has no tolerance at all**).

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
| Confirm `DOCKER_TWISTLOCK_TAG` is in the rendered cfg | `grep DOCKER_TWISTLOCK_TAG /opt/prisma-install/twistlock.cfg` |
| Diff overlaid cfg vs shipped original | `diff /opt/prisma-install/twistlock.cfg.orig /opt/prisma-install/twistlock.cfg` |
| Confirm registries.conf has the twistlock alias | `grep -A1 '^\[aliases\]' /etc/containers/registries.conf` |
| List images podman has loaded | `podman images --format '{{.Repository}}:{{.Tag}}'` |
| Run twistlock.sh with bash tracing | `bash -x /opt/prisma-install/twistlock.sh -s console 2>&1 \| tail -60` |
| Test Nexus reachability from the EE (not the VM) | `curl -ksSI {{ nexus_base_url }}` run **inside** the execution environment |
