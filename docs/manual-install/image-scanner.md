# Manual install — Image Scanner (standalone Defender) VM

Greenfield, air-gapped image-scanner host on RHEL 8.10 with docker-ce
+ kernel FIPS. Runs a standalone Prisma Defender container registered
to the Console; the Console then drives registry / on-schedule image
scans through this Defender.

This doc is a command-by-command operator script. The equivalent
Ansible coverage lives in `roles/rhel_baseline`, `roles/fips_enable`,
`roles/docker_stage`, `roles/docker_config`, and (Phase 3, not yet
shipped) `roles/prisma_defender`.

## Prerequisites

### Hardware
- RHEL 8.10 VM, 4 vCPU, 8 GiB RAM.
- 200 GiB local storage.
- 50 GiB NFS mount at `/var/lib/twistlock-backup` (shared with Console
  backup path; this VM is a reader only, for DR-drill purposes).

### Operator artefacts (staged to `/root/prisma/`)
- **docker-ce RPMs** (air-gapped — operator-staged):
  - `docker-ce-28.3.3-1.el8.x86_64.rpm` + `.sha256`
  - `docker-ce-cli-28.3.3-1.el8.x86_64.rpm` + `.sha256`
  - `containerd.io-1.7.27-1.el8.x86_64.rpm` + `.sha256`
- `prisma_cloud_compute_edition_34_01_126.tar.gz` + `.sha256` (same
  tarball as Console; we use `linux/twistcli` from inside it).

### Network
- Console FQDN reachable on port 8084/tcp (Defender websocket) and
  8083/tcp (API, for install-token retrieval).
- Internal container registry (if the Defender image must be pulled
  from it rather than side-loaded).
- Internal DNS, NTP, SIEM reachable.
- No public-internet egress.

### Console pre-reqs
- Console installed, running, reachable.
- A **Defender install token** (single-use, short-lived). Obtain via:
  Console UI → Manage → Defenders → Deploy → Deployment settings →
  *Copy install token*. Alternatively via API with a service-account
  token:
  ```bash
  curl -sk -H "Authorization: Bearer $CONSOLE_API_TOKEN" \
    https://prisma-console.internal:8083/api/v1/settings/defender-install-token
  ```

### Secrets
- Defender install token: ephemeral, single-use, do not persist.
- Any image-scan registry auth tokens (added later via Console UI).

## Phase 1 — VM provisioning `[M]`

Out of scope for this doc. Produce a RHEL 8.10 VM sized per
Prerequisites; root SSH from the operator workstation; register to
internal Satellite; `dnf -y update` (while network to Satellite is
still available); reboot.

## Phase 2 — Kernel FIPS `[A/M]`

Identical to Console Phase 2:

```bash
fips-mode-setup --enable
systemctl reboot

# After reboot:
fips-mode-setup --check
cat /proc/sys/crypto/fips_enabled
update-crypto-policies --show
```

## Phase 3 — Host hardening baseline `[A]`

Same as Console Phase 3 **except**:

- **Do not install podman.** Docker-ce and podman must not coexist on
  the same host (design choice for operational clarity).
- **Do not open Console ports (8083, 8084).** The scanner has no
  inbound services beyond SSH.

```bash
# SELinux
sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
setenforce 1

# Core packages (NO podman, NO container-specific tooling yet)
dnf -y install lvm2 xfsprogs chrony firewalld \
  policycoreutils-python-utils openssl jq tar gzip logrotate rsyslog audit curl

# Core services
systemctl enable --now chronyd firewalld auditd rsyslog

# chrony, sshd hardening, sysctl, journald — same bodies as console.md Phase 3

# Firewalld — SSH only
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

**Verify:** same as Console Phase 3, minus the 8083/8084 expectation.

## Phase 4 — Data volume `[A]`

Sized for image-layer pull cache + Defender local scratch. 200 GiB on
`/dev/sdb` (or whatever data disk the VM has).

```bash
pvcreate /dev/sdb
vgcreate vg_scanner /dev/sdb
lvcreate -l 100%FREE -n lv_data vg_scanner
mkfs.xfs -L scanner /dev/vg_scanner/lv_data

# Docker default graph root:
mkdir -p /var/lib/docker
echo "/dev/vg_scanner/lv_data /var/lib/docker xfs defaults,noatime,inode64 0 0" >> /etc/fstab
mount /var/lib/docker

# SELinux — docker already sets container_file_t on /var/lib/docker/overlay2
# once the daemon starts. No preemptive sefcontext needed.
```

## Phase 5 — docker-ce install `[A]`

```bash
mkdir -p /opt/docker-install
install -m 0600 /root/prisma/docker-ce-28.3.3-1.el8.x86_64.rpm         /opt/docker-install/
install -m 0600 /root/prisma/docker-ce-cli-28.3.3-1.el8.x86_64.rpm     /opt/docker-install/
install -m 0600 /root/prisma/containerd.io-1.7.27-1.el8.x86_64.rpm     /opt/docker-install/
install -m 0644 /root/prisma/docker-ce-28.3.3-1.el8.x86_64.rpm.sha256       /opt/docker-install/
install -m 0644 /root/prisma/docker-ce-cli-28.3.3-1.el8.x86_64.rpm.sha256   /opt/docker-install/
install -m 0644 /root/prisma/containerd.io-1.7.27-1.el8.x86_64.rpm.sha256   /opt/docker-install/

cd /opt/docker-install
for rpm in docker-ce docker-ce-cli containerd.io; do
  sha256sum -c "${rpm}"*.sha256       # each must print OK
done

# Confirm podman is NOT present (mutex invariant)
rpm -q podman && { echo "ERROR: podman present, aborting"; exit 1; }

dnf -y install --nogpgcheck \
  ./containerd.io-1.7.27-1.el8.x86_64.rpm \
  ./docker-ce-cli-28.3.3-1.el8.x86_64.rpm \
  ./docker-ce-28.3.3-1.el8.x86_64.rpm

docker --version                      # Docker version 28.3.3, build …
```

## Phase 6 — docker daemon configuration `[A]`

```bash
mkdir -p /etc/docker
cat >/etc/docker/daemon.json <<'EOF'
{
  "storage-driver": "overlay2",
  "log-driver": "journald",
  "log-opts": { "max-size": "100m", "max-file": "3" },
  "live-restore": true,
  "exec-opts": ["native.cgroupdriver=systemd"],
  "default-ulimits": {
    "nofile": { "Name": "nofile", "Soft": 65535, "Hard": 65535 },
    "nproc":  { "Name": "nproc",  "Soft": 4096,  "Hard": 4096 }
  }
}
EOF

mkdir -p /etc/systemd/system/docker.service.d
cat >/etc/systemd/system/docker.service.d/override.conf <<'EOF'
[Service]
Restart=on-failure
RestartSec=5s
CPUAccounting=yes
MemoryAccounting=yes
IOAccounting=yes
TasksMax=8192
EOF

systemctl daemon-reload
systemctl enable --now docker.service

# Smoke
docker info --format '{{.Driver}} {{.CgroupDriver}} {{.SecurityOptions}}'
#    expect: overlay2 systemd [name=seccomp,profile=builtin name=selinux]
```

### If the Defender image must come from an internal registry with a private CA

```bash
mkdir -p /etc/docker/certs.d/registry.internal
install -m 0644 /root/prisma/registry-ca.crt /etc/docker/certs.d/registry.internal/ca.crt
systemctl restart docker
```

## Phase 7 — Stage `twistcli` `[A]`

The scanner needs `twistcli` for the Defender install. Same tarball
as Console; extract it locally.

```bash
mkdir -p /opt/prisma-install
install -m 0600 /root/prisma/prisma_cloud_compute_edition_34_01_126.tar.gz        /opt/prisma-install/
install -m 0644 /root/prisma/prisma_cloud_compute_edition_34_01_126.tar.gz.sha256 /opt/prisma-install/

cd /opt/prisma-install
sha256sum -c prisma_cloud_compute_edition_34_01_126.tar.gz.sha256
tar -xzf prisma_cloud_compute_edition_34_01_126.tar.gz

install -m 0755 \
  prisma_cloud_compute_edition_34_01_126/linux/twistcli \
  /usr/local/bin/twistcli

twistcli --version                    # 34.01.126
```

## Phase 8 — Defender install `[M]`

**This step uses an ephemeral install token — do not persist it in
shell history.**

```bash
# Read the token from a file (created by the operator just before
# running this command), rather than pasting it into the shell.
read -s -r TOKEN < /root/prisma/defender-install-token
CONSOLE_ADDR="https://prisma-console.internal:8084"
unset HISTFILE    # defence in depth against history logging

twistcli defender install standalone \
  --address "$CONSOLE_ADDR" \
  --install-token "$TOKEN" \
  --cluster-address "$CONSOLE_ADDR" 2>&1 | tee /root/twistcli-defender-install.log

# Flag names to confirm against `twistcli defender install standalone --help`.
# See docs/manual-validation-guide.md — the definitive flag set has not yet
# been observed on a lab VM; the above is the expected form.

# Scrub the token file
shred -u /root/prisma/defender-install-token
```

Expected outcome: a `twistlock-defender*` systemd unit is created,
docker pulls / loads the Defender image, the container starts, and
it registers with the Console. Exact path and systemd unit name are
part of Phase 3 manual validation (see `docs/manual-validation-guide.md`).

**Verify:**

```bash
systemctl list-units 'twistlock-defender*'
systemctl status twistlock-defender*
docker ps                             # defender container, Up
docker logs --tail 50 $(docker ps -qf 'name=twistlock')

# In the Console UI: Manage → Defenders — the new Defender appears with
# this hostname, version 34.01.126, status Connected.
curl -sk -H "Authorization: Bearer $CONSOLE_API_TOKEN" \
  "$CONSOLE_ADDR_HTTPS/api/v1/defenders" | jq '.[] | select(.hostname=="'"$(hostname -f)"'")'
```

## Phase 9 — Registry scan scope (optional, Console-side)

In the Console UI: Defend → Vulnerabilities → Images → Registry
settings. Add the internal registry, credentials, scan schedule,
scope (repos / tags). Target this scanner Defender by group.

## Phase 10 — Logging + monitoring `[A]`

- **Journald + rsyslog → SIEM:** same setup as Console; forward
  `docker.service` and `twistlock-defender*.service` unit logs.
- **node_exporter:** optional; if in use, open firewalld 9100/tcp
  from the Prometheus source CIDR.

## Verification checklist

```bash
# FIPS
cat /proc/sys/crypto/fips_enabled           # 1
update-crypto-policies --show               # FIPS

# Mutex
rpm -q podman                               # "not installed"
rpm -q docker-ce docker-ce-cli containerd.io   # all "installed"

# Docker runtime
systemctl is-active docker
docker info --format '{{.Driver}} {{.CgroupDriver}}'
docker run --rm registry.internal/hello:latest  # optional reachability smoke

# Defender
systemctl list-units 'twistlock-defender*'
docker ps
# In Console UI, Defender shows Connected, Communication = OK.

# No unexpected inbound ports
firewall-cmd --list-all                     # ssh only (+ 9100 if node_exporter)
```

## Troubleshooting

- **`twistcli defender install` exits with "connection refused" to
  Console.** Check firewall egress from scanner to Console:8084.
  `curl -sk $CONSOLE_ADDR_HTTPS/api/v1/_ping` should return HTTP 200.
- **Defender container crashes on start.** `docker logs
  twistlock-defender`. Common causes: Console TLS chain not trusted
  (need CA cert mounted), install token already used (regenerate),
  time drift > 30s (chrony).
- **FIPS rejects Defender TLS handshake.** Capture the exact error
  from `docker logs`. Palo Alto TAC may need to confirm whether the
  shipped Defender image in v34.01.126 is built against a FIPS-validated
  Go toolchain. See [`docs/fips-feasibility.md`](../fips-feasibility.md)
  § Layer 3.
- **SELinux denials.** `ausearch -m AVC -ts recent`. The Defender
  container needs `container_file_t` on the docker graph dir — this
  is default. If denials persist, confirm `semanage boolean -l |
  grep container_manage_cgroup` is `on`.

## Cross-references

- [Ansible — `docker_stage`](../../roles/docker_stage/README.md)
- [Ansible — `docker_config`](../../roles/docker_config/README.md)
- [Manual validation guide](../manual-validation-guide.md) — run this
  before committing the future `prisma_defender` role logic.
- [FIPS feasibility](../fips-feasibility.md) § Layer 3
