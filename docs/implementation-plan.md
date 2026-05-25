# Implementation Plan — Multi-env, Defender/Sandbox, DR, Memory, Manual Docs

Scope for the expansion described in `New-Tasks.md`. This plan was designed
after three rounds of scoping questions; the user decisions recorded below
are load-bearing — revisit them before changing direction.

## User-scoped decisions

- **Task 1 inventory:** structure all 5 envs, populate only `dev/`. Other envs
  get commented template `hosts.yml`.
- **Task 1 FIPS on scanner/sandbox:** research + implement if feasible;
  findings recorded in `docs/fips-feasibility.md` (Phase 2 deliverable).
- **Task 1 Image Scanner:** standalone Prisma Defender container, auto-wired
  via Console API at install time (install token vaulted).
- **Task 1 Sandbox CI registration:** Ansible preps host only; CloudBees CI
  node registration stays manual.
- **Task 1 Docker RPMs:** operator-staged in `files/docker/` with `.sha256`
  manifests (mirrors existing `prisma_stage` pattern).
- **Task 2 DR drill:** restore-verify on secondary (stop → restore from NFS →
  verify → report). Primary curl-push backup flow stays as-is; NFS is the
  read side for the drill only.
- **Task 2 backup dir:** change `prisma_backup_dir` default from
  `/var/backups/prisma` → `/var/lib/twistlock-backup` fleet-wide.
- **Task 3 memory:** three layers — container `--memory` flags, sysctl,
  systemd `MemoryHigh=90%`/`MemoryMax=95%` in `override.conf.j2`.
- **Task 4 docs:** one doc per component under `docs/manual-install/`.

## File layout changes

### Inventory (Phase 1 — this change)

```
inventories/                         # one directory per environment
  dev/                               # fully populated (placeholder IPs/FQDNs)
    hosts.yml                        # prisma_console/scanner/sandbox × primary + secondary
    group_vars/
      all.yml                        # env-wide
      prisma_all.yml                 # all host classes
      prisma_console.yml             # console class (de-vaulted)
      prisma_scanner.yml             # scanner class
      prisma_sandbox.yml             # sandbox class
      primary.yml                    # live site (ansible_group_priority: 10)
      secondary.yml                  # DR target (is_dr_target: true)
    host_vars/                       # per-host FQDN / data-device overrides
  preproduction/                     # same shape
  production/                        # same shape
```

Superseded (AAP restructure): `group_vars/` now lives **inside** each
environment at `inventories/<env>/group_vars/` (dev, preproduction,
production), loaded adjacent to `-i inventories/<env>/hosts.yml`. Each host
sits on a role group (`prisma_*`) and a site group (`primary`/`secondary`).
The old root-level `group_vars/` and singular `inventory/` tree were removed.

### New roles (Phase 2+)

- `docker_stage` — mirrors `prisma_stage`. Copies docker-ce-28.3.3,
  docker-ce-cli-28.3.3, containerd.io-1.7.27 RPMs + `.sha256` manifests,
  verifies, `dnf` local-install.
- `docker_config` — `/etc/docker/daemon.json`, storage driver, journald,
  systemd override with memory limits.
- `prisma_defender` — stages twistcli, fetches Defender install token via
  Console API (vaulted), installs and registers standalone Defender
  container.
- `prisma_sandbox` — java-17, CI user + ssh keys, sandbox detonation
  tmpdir, twistcli staging, docker group membership.
- `prisma_restore` — DR drill: stop secondary console, pick latest tarball
  from NFS, restore via API or data-dir unpack, start, verify health.

### New playbooks (Phase 2+)

- `11-docker.yml` — hosts `prisma_scanner:prisma_sandbox`
- `21-defender.yml` — hosts `prisma_scanner` (runs after Console Phase 9)
- `22-sandbox.yml` — hosts `prisma_sandbox`
- `40-dr-drill.yml` — hosts `prisma_console:&secondary`

### Modified roles (Phase 2+)

- `prisma_backup/defaults/main.yml`: `prisma_backup_dir: /var/lib/twistlock-backup`
- `prisma_systemd/templates/override.conf.j2`: add `MemoryHigh`, `MemoryMax`, `MemorySwapMax=0`
- `prisma_systemd/defaults/main.yml`: add memory percent vars
- `rhel_baseline/defaults/main.yml`: extend `rhel_baseline_sysctl` with
  `vm.overcommit_memory`, `vm.dirty_ratio`, `vm.dirty_background_ratio`;
  add THP mode control.

### New docs

- `docs/fips-feasibility.md` — docker-ce/Defender/twistcli FIPS research
- `docs/manual-install/{console,image-scanner,sandbox}.md`
- `docs/dr-drill.md`

## Variable precedence

Lowest → highest:
1. role defaults
2. `inventories/<env>/group_vars/all.yml` (env-wide)
3. `inventories/<env>/group_vars/prisma_all.yml` (all host classes)
4. `inventories/<env>/group_vars/prisma_{console,scanner,sandbox}.yml` (role class)
5. `inventories/<env>/group_vars/{primary,secondary}.yml` (per-site, ansible_group_priority: 10)
6. `inventories/<env>/host_vars/<host>.yml` (host)

## New vault variables

- `prisma_defender_install_token` — Defender install token (Console API)
- `prisma_console_api_user` / `prisma_console_api_password` — used by scanner
  bootstrap to fetch install token
- `prisma_nfs_server` / `prisma_nfs_export` — may be per-DC, non-secret;
  vault only if deemed sensitive

## Phased rollout

1. **Phase 1** — inventory restructure (no behavior change). Shipped as
   commit `91c8fbc`.
2. **Phase 2** — `docker_stage` + `docker_config` roles +
   `11-docker.yml`; `rhel_baseline` split so the role runs against
   `prisma_all`; `00-baseline.yml` retargeted to `prisma_all` for
   baseline+FIPS, `prisma_console` for storage; `11-docker.yml` wired
   into `site.yml`; FIPS feasibility doc with the twistcli v34.01.126
   findings.
3. **Phase 3** — **manual install on a lab scanner VM first**
   (`twistcli defender install standalone`) to confirm runtime
   behaviour under FIPS, per the checklist in
   `docs/fips-feasibility.md#phase-3-manual-validation-checklist`.
   **Then** `prisma_defender` role + `21-defender.yml` automating the
   confirmed path.
4. **Phase 4** — manual `twistcli sandbox` test on a lab sandbox VM,
   then `prisma_sandbox` role + `22-sandbox.yml`.
5. **Phase 5** — memory enforcement (Task 3). Shipped. Three layers:
   - Sysctl: `vm.overcommit_memory=1`, `vm.dirty_ratio=15`,
     `vm.dirty_background_ratio=5` (added to `rhel_baseline_sysctl`).
   - THP: `disable-thp.service` systemd oneshot writes `never` to
     `/sys/kernel/mm/transparent_hugepage/{enabled,defrag}` at boot.
   - Cgroup: `MemoryHigh=90%` / `MemoryMax=95%` / `MemorySwapMax=0`
     on `twistlock.service` (prisma_systemd) and `docker.service`
     (docker_config). Per-container `--memory` caps still land in
     Phase 3 / Phase 4 at Defender / sandbox install time.
6. **Phase 6** — `prisma_restore` + `40-dr-drill.yml`;
   `prisma_backup_dir` changed from `/var/backups/prisma` →
   `/var/lib/twistlock-backup` in `group_vars/prisma_console.yml` +
   `docs/runbook.md`. Shipped. Uses `twistcli restore`; exact flag
   shape to be confirmed in Phase 3 manual validation (noted in
   `roles/prisma_restore/README.md`). Safety interlock
   (`prisma_restore_confirmed=true`) prevents accidental primary wipe.
   Operator guide: `docs/dr-drill.md`.
7. **Phase 7** — manual-install docs + validation guide. Shipped
   independently of Phases 3–6 because docs do not depend on lab
   access:
   - `docs/manual-install/console.md`
   - `docs/manual-install/image-scanner.md`
   - `docs/manual-install/sandbox.md`
   - `docs/manual-validation-guide.md` (for the eventual Phase 3
     runtime-behaviour validation)

## Verification

- `yamllint` + `ansible-lint` + `ansible-playbook --syntax-check` stay green
  at every phase.
- `molecule/default` extended with scanner + sandbox hosts where feasible
  (Phase 2+).
- `molecule/full` (docker) picks up the docker-based roles.
- DR drill verified in `molecule/full` with NFS mock (local bind mount).

## Risks

1. **docker-ce + podman coexistence:** separate host classes enforce this;
   add pre-task assert in `docker_config` that podman is absent and vice-versa.
2. **FIPS under docker-ce / Defender / twistcli is unverified** —
   `docs/fips-feasibility.md` to decide per component.
3. **NFS on `/var/lib/twistlock-backup`** is distinct from data dir
   `/var/lib/twistlock` (local XFS) — invariant preserved, but add verify.yml
   assertion.
4. **`21-defender.yml` needs Console up** — pre-task `_ping` gate.
5. **95% MemoryMax is aggressive** — OOM-restart risk. `MemoryHigh=90%`
   (soft throttle) + `MemoryMax=95%` (hard cap) documented in role README.
6. **Lint skips** (`fqcn[action-core]`, `no-changed-when`,
   `var-naming[no-role-prefix]`): new roles inherit same justified skip
   pattern.

## Open questions

1. Defender install method — `twistcli defender install docker` vs REST
   `/api/v1/defenders/install-bundle`? Confirm against v34.01.126 docs in
   tarball.
2. Prisma v34 REST restore (`POST /api/v1/backups/restore`) vs UI-only?
   If UI-only, `prisma_restore` unpacks tarball directly with Console
   stopped.
3. Does sandbox share twistcli version with scanner? Single
   `twistcli_version` var assumed.
4. NFS mount source / options per DC — confirm export-path convention.
5. DR drill cleanup — restore → verify → VMware-snapshot-revert? Out of
   scope for Ansible; documented as manual.
