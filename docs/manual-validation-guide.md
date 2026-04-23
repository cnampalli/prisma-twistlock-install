# Phase 3 manual validation — twistcli runtime behaviour

Companion to `docs/fips-feasibility.md`. This doc is the step-by-step
form of the Phase 3 manual-validation checklist. Run it against a lab
scanner VM once one is available, record results in the
`Phase 3 validation results` section of `fips-feasibility.md`, and
commit.

## Goal

Determine, by direct observation, whether `twistcli defender install
standalone` hardcodes `docker` under the hood or auto-detects the host
runtime. This answers the load-bearing question behind Phase 3's
`prisma_defender` role: do we commit to docker-ce on scanner hosts, or
is podman-everywhere viable?

## Prerequisites

- **Lab scanner VM — docker path.** RHEL 8.10, Phase 2 converged
  against it (docker-ce-28.3.3 installed, FIPS on, no podman). The
  `11-docker.yml` playbook is the way to get here; use a populated
  `inventory/dev/hosts.yml` or a one-off inventory with a single host
  in `prisma_scanner`.
- **Lab scanner VM — podman path.** A second RHEL 8.10 VM with **no
  docker-ce installed** and **podman installed** from the RHEL base
  repo. Easiest way to produce it: apply `rhel_baseline` with
  `rhel_baseline_container_packages: [podman]` plus `fips_enable`, but
  do not run `docker_stage`.
- A working Prisma Console reachable from both lab VMs (v34.01.126,
  the same build as the operator tarball).
- A Defender install token from the Console
  (`Manage → Defenders → Deploy → Deployment settings → Install scripts`,
  or via API with a service account token).
- The extracted tarball reachable on both VMs at a known path (below,
  `$EXTRACT_ROOT`).

## Commands to run and outputs to capture

### 1. Full subcommand surface

```bash
EXTRACT_ROOT="/opt/prisma-install/prisma_cloud_compute_edition_34_01_126"
"$EXTRACT_ROOT"/linux/twistcli defender install standalone --help 2>&1 | tee /tmp/twistcli-standalone-help.txt
"$EXTRACT_ROOT"/linux/twistcli sandbox --help 2>&1 | tee /tmp/twistcli-sandbox-help.txt
```

Paste both files verbatim into the `Phase 3 validation results`
section of `fips-feasibility.md`. Look specifically for any of:

- `--runtime`, `--container-runtime`, `--engine`
- `--docker-socket`, `--sock`, `--socket`
- `--rootless`
- Any path that mentions `docker.sock` or `/run/containerd`
- SELinux label flags (`--selinux-enabled`, etc.)

### 2. Docker-ce host — run the install

On the **docker-ce VM** (Phase 2 converged):

```bash
TOKEN="<defender-install-token-from-console>"
CONSOLE_ADDR="https://prisma-console.internal:8084"

"$EXTRACT_ROOT"/linux/twistcli defender install standalone \
  --address "$CONSOLE_ADDR" \
  --install-token "$TOKEN" \
  --cluster-address "$CONSOLE_ADDR" 2>&1 | tee /tmp/twistcli-standalone-docker.log
```

Substitute flag names if the `--help` above differed. Capture:

- Exit code.
- Every shell command twistcli invoked (if it prints them). If not,
  watch with `strace` or `auditd` in a second terminal:
  `execsnoop -n twistcli` (requires bcc-tools) or
  `strace -f -e trace=execve -o /tmp/twistcli-execve.log <command>`.
- Any generated artefacts on disk. The common places to check:
  - `/etc/systemd/system/twistlock-defender*.service`
  - `/etc/twistlock/` (Prisma's default conf dir)
  - `/var/lib/twistlock-defender/`
  - `/opt/twistlock/`
- `systemctl status twistlock-defender` (or whatever unit was
  created) — does the `ExecStart=` line say `docker run`,
  `podman run`, or an absolute path to a container binary?
- `docker ps` — is a Defender container running?
- In the Console UI (`Manage → Defenders`), does the new Defender
  appear with hostname, version, and status `Connected`?

### 3. Podman-only host — run the same install

On the **podman-only VM** (no docker-ce):

```bash
"$EXTRACT_ROOT"/linux/twistcli defender install standalone \
  --address "$CONSOLE_ADDR" \
  --install-token "$TOKEN" \
  --cluster-address "$CONSOLE_ADDR" 2>&1 | tee /tmp/twistcli-standalone-podman.log
```

Three possible outcomes — all are informative:

- **"docker: command not found"** or **"cannot connect to daemon at
  unix:///var/run/docker.sock"**: docker is required, podman path is
  dead, Phase 3 plan is docker-ce on scanner hosts.
- **Install succeeds and a Defender container runs under podman**:
  docker-only premise in `New-Tasks.md` is wrong; podman-everywhere
  is viable. Plan a later refactor to drop docker-ce.
- **Install succeeds but the generated systemd unit references
  `docker`** (e.g. `ExecStart=/usr/bin/docker run …`): podman is
  partially compatible; we'd need a `docker -> podman` symlink or a
  `podman-docker` package. Borderline — record the exact unit file.

### 4. Inspect the generated systemd unit

Whichever host succeeds, capture the generated artefact verbatim:

```bash
cat /etc/systemd/system/twistlock-defender*.service
```

If there is no systemd unit, check:

```bash
ls -la /etc/twistlock/ /opt/twistlock/ /var/lib/twistlock-defender/ 2>&1
find / -xdev -name 'twistlock*' -newer /tmp/twistcli-standalone-docker.log 2>/dev/null
```

### 5. Runtime attribution

On the docker-ce host where the Defender is running:

```bash
docker inspect --format '{{.HostConfig.PidMode}} {{.HostConfig.NetworkMode}} {{.HostConfig.Privileged}}' <defender-container>
docker inspect --format '{{json .Config.Labels}}' <defender-container>
```

Record the `--privileged`, `--pid=host`, `--net=host`, and any custom
label set. The `prisma_defender` Ansible role (Phase 3 automation) will
need to reproduce these exactly.

### 6. FIPS attribution

Inside the running Defender container:

```bash
docker exec -it <defender-container> /bin/sh -c '
  echo "--- /proc/sys/crypto/fips_enabled:"; cat /proc/sys/crypto/fips_enabled
  echo "--- openssl version:";              openssl version 2>/dev/null || echo none
  echo "--- crypto policy file:";           ls -la /etc/crypto-policies 2>/dev/null
'
```

Kernel-inherited FIPS should show `1` inside the container. If
openssl is present, its version string usually includes `fips`.

## Record results

Append to `docs/fips-feasibility.md` under the
`### Phase 3 validation results` placeholder. Minimum content:

- Date, operator, lab VM identifiers.
- Which of the three outcomes (docker-required / podman-works /
  partial) the podman-only run produced.
- Exact paths + contents of the generated systemd unit(s).
- Defender registration confirmation in the Console UI (screenshot
  optional but useful).
- `docker inspect` label/flag dump for the `prisma_defender` role to
  reproduce.
- FIPS attribution findings.

Once recorded, commit the FIPS doc update. `prisma_defender` role
design in Phase 3 then proceeds on evidence rather than inference.

## When this can be skipped

It cannot — Phase 3's `prisma_defender` role needs either the
`docker run` flags twistcli uses or the podman-compat confirmation.
Without one of those, the role is guesswork. If lab access is
unavailable long-term, alternative paths are (a) engage Palo Alto
TAC for an authoritative answer, (b) build the Ansible role by
translating the generated systemd unit after a human has run the
install once, even if not on our lab — in either case, the "record
results" artefact is what unblocks Phase 3.
