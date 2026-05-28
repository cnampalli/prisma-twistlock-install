# Manual install — Sandbox (twistcli detonation) VM

Greenfield, air-gapped sandbox host on RHEL 8.10 with docker-ce +
kernel FIPS + java-17 + a local CI agent user. CloudBees CI Jenkins
connects to this host as an agent node; pipelines invoke
`twistcli sandbox` on it to detonate candidate images and capture
behavioural signals.

This doc is a command-by-command operator script. The equivalent
Ansible coverage lives in `roles/rhel_baseline`, `roles/fips_enable`,
`roles/docker_stage`, `roles/docker_config`, and (Phase 4, not yet
shipped) `roles/prisma_sandbox`.

## What this host is not

- Not a Defender. Nothing registers with the Console from here.
- Not a persistent scanner. The sandbox is invoked per-pipeline-run
  and the detonation environment is torn down each time.
- Not a Jenkins controller. Only a CI agent node; the controller runs
  elsewhere in the CloudBees CI deployment.

## Prerequisites

### Hardware
- RHEL 8.10 VM, 4 vCPU, 8 GiB RAM.
- 200 GiB local storage (image-layer cache + detonation scratch).
- 50 GiB NFS mount at `/var/lib/twistlock-backup` (optional —
  provisioned for consistency with the other VM types; sandbox
  doesn't actively use it).

### Operator artefacts (staged to `/root/prisma/`)
- docker-ce RPMs (same three as image-scanner Phase 5).
- `prisma_cloud_compute_edition_34_01_126.tar.gz` + `.sha256` (for
  `twistcli` extraction).
- java-17-openjdk RPMs (or internal Satellite reachability for `dnf`).
- Public key(s) for the CI agent user to receive from the CloudBees
  CI controller. These are the keys the controller will present when
  connecting; get them from your CloudBees CI admin.

### Network
- Console reachable on 8083/tcp (twistcli needs to authenticate to
  the Console for sandbox analysis, even if the sandbox itself
  doesn't run a persistent Defender). Flag that to TAC if in doubt
  — some twistcli subcommands need Console coordinates.
- CloudBees CI controller reachable on 50000/tcp (or whatever the
  JNLP / WebSocket port is).
- Internal container registry (image under test is pulled from here).
- Internal DNS, NTP, SIEM reachable.
- No public-internet egress.

### Secrets
- Console service-account token for twistcli — scoped to
  vuln-scanning + sandbox. Stored in the CI agent user's home,
  readable only by that user (mode 0600).
- The CI agent user's ssh `authorized_keys` — not a secret per se, but
  install exactly the keys CloudBees CI admin provides.

## Phase 1 — VM provisioning `[M]`

Out of scope. Standard RHEL 8.10 build, registered to Satellite,
`dnf -y update`, reboot.

## Phase 2 — Kernel FIPS `[A/M]`

```bash
fips-mode-setup --enable
systemctl reboot

# After reboot:
cat /proc/sys/crypto/fips_enabled              # 1
update-crypto-policies --show                  # FIPS
```

## Phase 3 — Host hardening baseline `[A]`

Same as image-scanner Phase 3: base packages **without podman**,
firewalld allowing SSH only (no inbound 8083/8084).

```bash
sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
setenforce 1

dnf -y install lvm2 xfsprogs chrony firewalld \
  policycoreutils-python-utils openssl jq tar gzip logrotate rsyslog audit curl

systemctl enable --now chronyd firewalld auditd rsyslog
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload

# chrony.conf, sshd hardening, sysctl, journald — same as
# console.md Phase 3 bodies.
```

## Phase 4 — Data volume `[A]`

```bash
pvcreate /dev/sdb
vgcreate vg_sandbox /dev/sdb
lvcreate -l 100%FREE -n lv_data vg_sandbox
mkfs.xfs -L sandbox /dev/vg_sandbox/lv_data

mkdir -p /var/lib/docker
echo "/dev/vg_sandbox/lv_data /var/lib/docker xfs defaults,noatime,inode64 0 0" >> /etc/fstab
mount /var/lib/docker
```

## Phase 5 — docker-ce install `[A]`

Identical to image-scanner Phase 5. `rpm -q podman` must fail;
sha256-verify the three RPMs; `dnf install --nogpgcheck` the trio.

## Phase 6 — docker daemon configuration `[A]`

Identical to image-scanner Phase 6 (daemon.json + systemd override,
enable + start docker). See that doc for the full bodies.

## Phase 7 — java-17-openjdk `[A]`

CloudBees CI agents require a Java runtime to run the remoting agent
JAR. Java 17 is the supported baseline for recent CloudBees CI
versions; confirm against your controller's release notes.

```bash
dnf -y install java-17-openjdk java-17-openjdk-devel
java -version                              # openjdk 17.…
update-alternatives --config java          # confirm 17 is default
```

## Phase 8 — Local CI agent user `[A]`

Create a dedicated non-root user the CloudBees CI controller will
connect as. The user needs a home, ssh authorized_keys, and — in
order to invoke docker / twistcli — membership in the `docker` group.

```bash
CI_USER="cagent"
CI_HOME="/home/$CI_USER"
CI_KEY_SRC="/root/prisma/cagent-authorized-keys"   # provided by CloudBees admin

useradd -m -d "$CI_HOME" -s /bin/bash "$CI_USER"
usermod -aG docker "$CI_USER"

install -d -m 0700 -o "$CI_USER" -g "$CI_USER" "$CI_HOME/.ssh"
install -m 0600 -o "$CI_USER" -g "$CI_USER" "$CI_KEY_SRC" "$CI_HOME/.ssh/authorized_keys"

# SELinux — restorecon on the new home dir
restorecon -R "$CI_HOME"

# Password-less sudo NOT granted — CI jobs should not need root on this host.
```

**Verify:** from the CloudBees CI controller host, `ssh
cagent@sandbox.internal java -version` should succeed without a
password.

## Phase 9 — Sandbox detonation scratch dir `[A]`

Dedicated tmpdir with SELinux context allowing the CI agent to
write and docker to read. Cleared between runs by the CI job.

```bash
install -d -m 0770 -o "$CI_USER" -g docker /var/lib/twistlock-sandbox

# Tell SELinux this path is for container content
semanage fcontext -a -t container_file_t '/var/lib/twistlock-sandbox(/.*)?'
restorecon -R /var/lib/twistlock-sandbox

# Optional: a tmpfiles.d rule to purge stale artefacts > 24h
cat >/etc/tmpfiles.d/prisma-sandbox.conf <<'EOF'
d /var/lib/twistlock-sandbox 0770 cagent docker 1d
EOF
systemd-tmpfiles --create /etc/tmpfiles.d/prisma-sandbox.conf
```

## Phase 10 — Stage `twistcli` for the CI user `[A]`

```bash
mkdir -p /opt/prisma-install
install -m 0600 /root/prisma/prisma_cloud_compute_edition_34_01_126.tar.gz        /opt/prisma-install/
install -m 0644 /root/prisma/prisma_cloud_compute_edition_34_01_126.tar.gz.sha256 /opt/prisma-install/
cd /opt/prisma-install
sha256sum -c prisma_cloud_compute_edition_34_01_126.tar.gz.sha256
tar -xzf prisma_cloud_compute_edition_34_01_126.tar.gz   # extracts flat into /opt/prisma-install/

install -m 0755 linux/twistcli /usr/local/bin/twistcli

# Validate
twistcli --version
twistcli sandbox --help | head -40
```

## Phase 11 — Console service-account token for the CI user `[M]`

Create a service-account user in the Console dedicated to sandbox
analysis (UI: Manage → Authentication → Users → Add user). Scope:
minimum needed for sandbox + image scanning.

Install the token into the CI user's home, mode 0600:

```bash
install -d -m 0700 -o "$CI_USER" -g "$CI_USER" "$CI_HOME/.prisma"
install -m 0600 -o "$CI_USER" -g "$CI_USER" \
  /root/prisma/cagent-console-token "$CI_HOME/.prisma/console-token"

# The CI job reads this file as `cagent` and passes it to twistcli
# via a flag like --token-file or via CONSOLE_TOKEN env var. Confirm
# the exact flag against `twistcli sandbox --help`.

shred -u /root/prisma/cagent-console-token
```

## Phase 12 — Smoke test the full pipeline invocation `[M]`

From a CloudBees CI job (or locally as the CI user), invoke sandbox
analysis against a known image in your internal registry:

```bash
sudo -iu "$CI_USER"                    # or run from the CI job directly

twistcli sandbox \
  --address "https://prisma-console.internal:8083" \
  --user "$CI_USER" \
  --password-file "$HOME/.prisma/console-token" \
  registry.internal/hello:latest 2>&1 | tee /tmp/sandbox-smoke.log
```

Substitute `--user` / `--password` / `--token` flags against what
`twistcli sandbox --help` shows. The sandbox subcommand pulls the
image, runs it in a contained detonation environment (via docker),
and reports behavioural observations back to the Console / to stdout.

**Verify:**

```bash
docker ps                              # transient containers spawned by twistcli
docker ps -a --filter status=exited --latest
# In the Console UI: Monitor → Runtime → Sandbox scans, a new entry
# for registry.internal/hello:latest.
```

## Phase 13 — CloudBees CI agent node registration `[M]`

Out of scope for Ansible. In the CloudBees CI controller UI:

1. Manage Jenkins → Nodes → New Node.
2. Name = sandbox VM's FQDN. Type = Permanent Agent.
3. Remote root directory = `/home/cagent/jenkins-agent`.
4. Launch method = "Launch agents via SSH".
5. Host = sandbox FQDN; Credentials = the key whose public half
   you installed in Phase 8.
6. Labels = `prisma-sandbox` (or your org's convention). Jobs that
   need detonation target this label.
7. Save; the controller connects; node status = online.

Confirm from the sandbox VM:

```bash
sudo -iu cagent pgrep -fa 'java.*agent.jar'   # running
```

## Phase 14 — Logging + monitoring `[A]`

- journald + rsyslog → SIEM: forward `docker.service` and the CI
  user's systemd user slice if you want job-level visibility.
- node_exporter: optional.

## Verification checklist

```bash
# FIPS
cat /proc/sys/crypto/fips_enabled           # 1

# Mutex
rpm -q podman                               # not installed
rpm -q docker-ce docker-ce-cli containerd.io   # installed

# Docker
systemctl is-active docker
docker info --format '{{.Driver}} {{.CgroupDriver}}'

# Java
java -version                               # openjdk 17.…

# CI user
id cagent
getent group docker | grep -q cagent        # in docker group
sudo -iu cagent ssh -o BatchMode=yes sandbox-self 'true'   # key login works

# twistcli
which twistcli
twistcli --version
sudo -iu cagent twistcli sandbox --help | head

# SELinux + sandbox dir
ls -Zd /var/lib/twistlock-sandbox
getenforce

# No unexpected inbound ports
firewall-cmd --list-all                     # ssh only
```

## Troubleshooting

- **`twistcli sandbox` complains about docker permission denied.**
  CI user isn't in the `docker` group yet — `getent group docker`
  must list them. Logout / login the shell after `usermod -aG`.
- **CloudBees CI says "agent is offline".** Controller → agent SSH
  key mismatch (check the agent log under `Manage Jenkins → Nodes
  → <node> → See log`), or firewalld blocking the agent.jar TCP
  back-channel (recent CloudBees CI uses 50000/tcp by default, but
  many deployments use WebSocket mode instead — confirm).
- **Image pull fails under FIPS** (TLS handshake error against the
  internal registry). Registry CA trust missing —
  `/etc/docker/certs.d/registry.internal/ca.crt` must be in place
  before `docker pull`.
- **twistcli sandbox runs but the Console UI shows nothing.**
  Token scope too narrow — the service account needs *Sandbox
  analysis* permission. Check in Console → Manage → Authentication
  → Users → (user) → Permissions.
- **SELinux denies on `/var/lib/twistlock-sandbox`.** `ausearch -m
  AVC -ts recent -f /var/lib/twistlock-sandbox`. Re-run
  `restorecon -R /var/lib/twistlock-sandbox`; if still denied,
  `semanage fcontext -l | grep twistlock-sandbox` should show the
  `container_file_t` rule from Phase 9.

## Cross-references

- [Ansible — `docker_stage`](../../roles/docker_stage/README.md)
- [Ansible — `docker_config`](../../roles/docker_config/README.md)
- [Image scanner manual install](image-scanner.md) — shares Phases
  2–6.
- [Manual validation guide](../manual-validation-guide.md) —
  `twistcli sandbox --help` confirmation is part of Phase 3
  validation.
- [FIPS feasibility](../fips-feasibility.md)
