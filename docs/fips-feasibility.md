# FIPS feasibility — scanner and sandbox VMs

**Status:** research draft (Phase 2). What the twistcli binary tells us
about v34.01.126 is captured below; what remains unknown is flagged
explicitly as "to verify in Phase 3 manual install."

## Scope

The Prisma Console runs in FIPS 140-2 mode end-to-end (rootful podman
on FIPS-enabled RHEL 8.10). Task 1.5.1 of `New-Tasks.md` asks whether
the two docker-based host classes — Image Scanner (standalone Prisma
Defender) and Sandbox (twistcli detonation agent) — can also run FIPS
end-to-end.

Three layers per host, each with independent FIPS properties:

1. **RHEL 8.10 kernel** in FIPS mode (`fips-mode-setup --enable`).
2. **docker-ce 28.3.3** engine + CLI + containerd.io 1.7.27.
3. **Palo Alto userland** — standalone Defender (scanner) or
   twistcli sandbox (sandbox VM).

## Authoritative source

The tarball (`prisma_cloud_compute_edition_34_01_126.tar.gz`) is
binaries + installer scripts; it does **not** contain an
`admin-guide/` directory. The authoritative capability reference for
what twistcli can and cannot do is the binary's own `--help` output.
Release-level documentation lives at `docs.prismacloud.io` (web, not
bundled). We've inspected `twistcli --help` and
`twistcli defender install --help` directly — findings below.

## twistcli v34.01.126 findings (confirmed from the binary)

- **Top-level commands:** `support | defender | console | images |
  hosts | intelligence | restore | serverless | tas | app-embedded |
  sandbox | clustered-db | waas | help`.
- **`defender install` subcommands:** `standalone | kubernetes |
  openshift`. **No** `docker` / `podman` subcommand — install is
  runtime-agnostic at the CLI level. The scanner VM uses `standalone`.
- **No runtime-selection flag** exists in the subcommand help we
  inspected. Whether `twistcli defender install standalone` autodetects
  the host runtime or shells out to a hardcoded `docker` CLI is **not
  determinable from `--help` alone** and is deferred to the Phase 3
  manual validation below.
- **`sandbox` is a top-level `twistcli` command** ("Runs a sandbox
  analysis of an image"). Sandbox functionality ships inside the
  `twistcli` binary; the sandbox VM needs `twistcli` + an OCI runtime
  twistcli can drive.
- **`twistcli restore`** exists ("Restores Console's database from a
  backup file"). Directly useful for Task 2 DR drill (Phase 6).

## Runtime choice: docker-ce, not podman (for now)

Rationale:

1. `New-Tasks.md` Task 1.4 pins `docker-ce-28.3.3` /
   `docker-ce-cli-28.3.3` / `containerd.io-1.7.27`.
2. Absence of a `--runtime` flag in `standalone --help` is ambiguous,
   not permissive — we can't confirm podman compatibility without
   testing on a real lab VM (Phase 3).
3. Docker-ce is the historical default for Palo Alto standalone
   Defender installs; the Phase 2 scaffolding must work day-one.
4. If Phase 3 manual testing shows podman works out of the box, we can
   revisit in a later refactor — low regret to start with docker-ce.

### Why docker-ce, not Mirantis MCR

Mirantis Container Runtime (formerly Docker Enterprise) is the
commercially FIPS 140-2 validated docker engine on RHEL. We're not
using MCR because the task spec explicitly pins docker-ce; our FIPS
boundary is therefore **kernel + Palo Alto userland**, not the docker
daemon TLS stack. Documented here so the tradeoff is on the record,
not a silent assumption.

## Layer 1 — RHEL FIPS kernel: no blocker

The existing `fips_enable` role enables `fips=1` via
`fips-mode-setup`, reboots, and asserts
`/proc/sys/crypto/fips_enabled == 1`. Host-agnostic, reused verbatim
for scanner and sandbox via the Phase 2 split of `00-baseline.yml`
(now targets `prisma_all`).

Benefits inherited by every host:

- FIPS-validated kernel crypto (openssl, libgcrypt).
- System-wide crypto policy `FIPS`.
- DRBG from a FIPS-validated RNG.

**Verification on a scanner/sandbox lab host:**

```bash
cat /proc/sys/crypto/fips_enabled       # expect: 1
update-crypto-policies --show           # expect: FIPS
openssl version                         # expect: OpenSSL 1.1.1k-fips or later
```

## Layer 2 — docker-ce FIPS posture: known not validated

**Finding (not an open question):** upstream docker-ce is not FIPS
140-2 validated at the daemon crypto layer. The daemon + CLI are Go
binaries; Docker Inc.'s standard builds do not use a FIPS-validated
crypto module (no BoringCrypto, no Go+OpenSSL toolchain). Mirantis MCR
is the commercially validated alternative; our operator chose
docker-ce (see above). Containers running on the host still use
**kernel** crypto for any call into libcrypto/openssl — so Defender
and twistcli's *own* crypto may still be FIPS-sourced — but the
docker daemon's registry TLS and API TLS are not.

Operational implication: in our air-gapped topology the docker daemon
rarely originates TLS outbound — registry pulls go through the
Defender / twistcli path, and the Defender container's TLS stack is
the certified crypto boundary (to confirm in Phase 3). If any flow
requires the daemon itself to perform FIPS-validated TLS, docker-ce
is the wrong choice and MCR or a Go+FIPS-toolchain rebuild would be
the remediation. No such flow is known today.

## Layer 3 — Palo Alto userland: verify in Phase 3

Based on the binary's help text, v34.01.126 does not ship a separate
`-fips` twistcli variant that I can see; the single `linux/twistcli`
is what the tarball provides. Whether that binary and the Defender
container image it deploys are built against Red Hat's Go FIPS
toolchain (and therefore inherit kernel FIPS automatically) is **to
confirm in Phase 3**. Most modern Go binaries produced by RHCP-aware
vendors link against the system crypto; absence of an `-fips` suffix
is typical for this pattern, not evidence against it.

Specific things to confirm in Phase 3 (see checklist below):

1. Does `twistcli` + Defender image behave cleanly when kernel FIPS is
   on? (i.e. no crypto policy rejection at install time.)
2. Does Palo Alto's Console gate any features behind a FIPS-variant
   check, or is `prisma_fips_enabled: true` sufficient?
3. Does the Defender image ship with a `FIPS` marker, or is the
   single image FIPS-capable by virtue of its build toolchain?

## Baseline refactor — completed as part of Phase 2

`rhel_baseline` previously installed `podman` fleet-wide and opened
Console ports 8083/tcp + 8084/tcp in firewalld — both Console-specific
and both blockers for running the role on scanner/sandbox. Phase 2
applied this refactor:

- `rhel_baseline_packages` (role default) no longer contains `podman`.
- New `rhel_baseline_container_packages` hook defaults to `[]`.
  `group_vars/prisma_console.yml` sets it to `[podman]`.
- `rhel_baseline_firewall_ports` default is `[]`;
  `group_vars/prisma_console.yml` pins the Console ports.
- `00-baseline.yml` runs `rhel_baseline` + `fips_enable` against
  `prisma_all`; `prisma_storage` stays scoped to `prisma_console`.
- `site.yml` now imports `11-docker.yml` between `10-podman.yml` and
  `20-prisma-stage.yml`.

## Recommended posture (post-Phase-2)

- **Scanner (standalone Defender):** kernel FIPS **on** (inherited via
  `fips_enable` on `prisma_all`). Defender installed via `twistcli
  defender install standalone …` (Phase 3). If that install completes
  cleanly under FIPS, posture is "kernel-FIPS + vendor-provided
  userland." If it fails, escalate to Palo Alto TAC.
- **Sandbox (twistcli):** kernel FIPS **on**. twistcli invoked
  ad-hoc from the CI pipeline. TLS surface is limited to Console auth
  + image registry pulls during detonation.
- **Neither host class** is a formal FIPS 140-2 compliance boundary
  today. Claim kernel-FIPS + defense-in-depth until Phase 3
  findings either upgrade or downgrade that.

## Phase 2 acceptance tests (runnable now)

Per host after Phase 2 converges:

1. `cat /proc/sys/crypto/fips_enabled` → `1` (scanner + sandbox
   inherit this from `00-baseline.yml` retargeting).
2. `update-crypto-policies --show` → `FIPS`.
3. `rpm -q podman` → not installed (scanner + sandbox only).
4. `docker --version` → `Docker version 28.3.3`.
5. `docker info` runs without error; storage driver is `overlay2`,
   cgroup driver is `systemd`, security options include `seccomp`
   and `selinux`.
6. No inbound firewall ports on scanner / sandbox beyond SSH.

## Phase 3 manual-validation checklist

Once Phase 2 has installed docker-ce and enabled FIPS on a lab
scanner VM, run these manually (results get recorded here under
"Phase 3 validation results" afterward):

1. `./linux/twistcli defender install standalone --help` — capture
   every option, especially `--address`, `--cluster-address`,
   `--token`, `--privileged`, `--monitor`, SELinux flags, and any
   option that names a runtime or socket path.
2. `./linux/twistcli sandbox --help` — confirm the option set, note
   what default runtime assumption it makes, and whether a socket
   path can be specified.
3. On a **separate podman-only VM** (no docker-ce installed), attempt
   `twistcli defender install standalone`. Record exactly what
   happens:
   - If it fails with `docker: command not found` or `cannot connect
     to daemon at /var/run/docker.sock`, docker is required.
   - If it proceeds and produces a running Defender container under
     podman, the docker-only premise in `New-Tasks.md` is false and
     we can plan a podman-everywhere refactor in a later phase.
4. On the docker-ce VM, capture the generated Defender install
   artefact (systemd unit file, compose file, or direct `docker run`
   script). If it hardcodes `docker run …`, document the exact path;
   if it's a systemd unit invoking a runtime abstractly, note
   whether `CONTAINER_HOST` / similar env vars can redirect it to
   podman's socket.
5. Record all findings below.

### Phase 3 validation results

_(empty — populate after Phase 3 manual install on lab VMs)_

## Known findings vs open questions

**Known (confirmed from twistcli binary):**

- `defender install` has no `docker`/`podman` subcommand.
- `defender install standalone` is the scanner path.
- `sandbox` is a built-in twistcli command.
- `restore` is a built-in twistcli command — useful for Task 2 DR drill.

**Open (deferred to Phase 3 manual install):**

1. Does `twistcli defender install standalone` hardcode docker under
   the hood, autodetect runtime, or support a `--runtime` equivalent
   we can't see in `--help`?
2. Does the shipped Defender image / twistcli binary link against a
   FIPS-validated Go crypto toolchain?
3. Does `twistcli sandbox` tolerate a podman docker-compat socket?
4. What is the install method for an *air-gapped* Defender — is the
   image pulled from the Console's API, or must the operator stage it
   under `files/` first?
