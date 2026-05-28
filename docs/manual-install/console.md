# Manual install — Prisma Cloud Compute Console

Greenfield, air-gapped Console install on RHEL 8.10 with rootful podman
and FIPS 140-2. Command-by-command operator script for running the full
build **without Ansible**.

For background (architecture, BoM, network flows, acceptance
tables per phase), the source of truth is [`docs/runbook.md`](../runbook.md).
This doc is the condensed "what do I type" companion.

## Prerequisites

### Hardware
- RHEL 8.10 VM, 8 vCPU, 16 GiB RAM.
- 80 GiB OS disk + 500 GiB data disk (thick, local SSD — **not NFS**).
- 50 GiB NFS mount for backups (Task 2; mounted at
  `/var/lib/twistlock-backup`).

### Operator artefacts (staged to `/root/prisma/`)
- `prisma_cloud_compute_edition_34_01_126.tar.gz` + `.sha256`
- Prisma Compute licence key file
- `console.crt`, `console.key`, `ca-chain.crt` (from internal CA)
- LDAPS server CA cert

### Network
- Internal DNS resolves the Console FQDN.
- Internal NTP, LDAPS, SIEM, Prometheus reachable.
- No public-internet egress.
- Internal Satellite / yum mirror reachable for `dnf` (or a
  pre-populated local dnf cache).

### Secrets
- Vault or ansible-vault for LDAPS bind password, licence, backup
  token. Never commit plaintext.

## Phase 1 — VM provisioning `[M]`

See [runbook §5 Phase 1](../runbook.md#phase-1--vm-provisioning-on-vsphere-m).
Sign-off: `hostnamectl` shows FQDN; `lsblk` shows `sda` + `sdb`
unpartitioned; `/etc/redhat-release` is 8.10.

## Phase 2 — Kernel FIPS `[A/M]`

```bash
fips-mode-setup --enable                      # Prints "FIPS mode will be enabled"
systemctl reboot
# After reboot:
fips-mode-setup --check                       # "FIPS mode is enabled."
cat /proc/sys/crypto/fips_enabled             # 1
update-crypto-policies --show                 # FIPS
dmesg | grep -i fips                          # no errors
```

## Phase 3 — Host hardening baseline `[A]`

```bash
# SELinux
sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config
setenforce 1

# Packages
dnf -y install podman lvm2 xfsprogs chrony firewalld \
  policycoreutils-python-utils openssl jq tar gzip logrotate rsyslog audit curl

# Core services
systemctl enable --now chronyd firewalld auditd rsyslog

# /etc/chrony.conf — replace pool lines with your internal NTP:
cat >/etc/chrony.conf <<'EOF'
server ntp1.internal iburst
server ntp2.internal iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
leapsectz right/UTC
logdir /var/log/chrony
EOF
systemctl restart chronyd

# sshd hardening (validate before reload)
sed -i 's/^#*PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^#*PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sshd -t && systemctl reload sshd

# Firewall
firewall-cmd --permanent --add-port=8083/tcp
firewall-cmd --permanent --add-port=8084/tcp
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload

# sysctl
cat >/etc/sysctl.d/90-prisma.conf <<'EOF'
vm.swappiness = 10
vm.max_map_count = 262144
fs.file-max = 2097152
net.core.somaxconn = 1024
net.ipv4.tcp_max_syn_backlog = 1024
EOF
sysctl --system

# journald caps
mkdir -p /etc/systemd/journald.conf.d
cat >/etc/systemd/journald.conf.d/10-prisma.conf <<'EOF'
[Journal]
Storage=persistent
SystemMaxUse=2G
SystemKeepFree=1G
MaxFileSec=1week
Compress=yes
ForwardToSyslog=yes
EOF
systemctl restart systemd-journald
```

**Verify:**
```bash
getenforce                           # Enforcing
systemctl is-active chronyd firewalld auditd rsyslog
firewall-cmd --list-all
sysctl vm.swappiness vm.max_map_count
journalctl --disk-usage
```

## Phase 4 — Data volume on LVM + XFS `[A]`

```bash
pvcreate /dev/sdb
vgcreate vg_twistlock /dev/sdb
lvcreate -l 100%FREE -n lv_data vg_twistlock
mkfs.xfs -L twistlock /dev/vg_twistlock/lv_data

mkdir -p /var/lib/twistlock
echo "/dev/vg_twistlock/lv_data /var/lib/twistlock xfs defaults,noatime,inode64 0 0" >> /etc/fstab
mount /var/lib/twistlock

semanage fcontext -a -t container_file_t '/var/lib/twistlock(/.*)?'
restorecon -R /var/lib/twistlock
```

**Verify:**
```bash
df -hT /var/lib/twistlock
findmnt /var/lib/twistlock             # xfs, noatime, NOT nfs
ls -Zd /var/lib/twistlock              # container_file_t
```

## Phase 5 — Podman configuration `[A]`

```bash
# /etc/containers/registries.conf — air-gapped, single internal registry
test -f /etc/containers/registries.conf && \
  cp /etc/containers/registries.conf /etc/containers/registries.conf.orig

cat >/etc/containers/registries.conf <<'EOF'
unqualified-search-registries = []

[[registry]]
location = "registry.internal"
insecure = false
blocked = false

[[registry]]
location = "docker.io"
blocked = true

[[registry]]
location = "quay.io"
blocked = true
EOF

# If the internal registry uses a private CA:
mkdir -p /etc/containers/certs.d/registry.internal
cp /root/prisma/registry-ca.crt /etc/containers/certs.d/registry.internal/ca.crt

# /etc/containers/storage.conf — overlay driver, data dir on our XFS
test -f /etc/containers/storage.conf && \
  cp /etc/containers/storage.conf /etc/containers/storage.conf.orig

cat >/etc/containers/storage.conf <<'EOF'
[storage]
driver = "overlay"
runroot = "/run/containers/storage"
graphroot = "/var/lib/containers/storage"

[storage.options]
additionalimagestores = []

[storage.options.overlay]
mountopt = "nodev,metacopy=on"
EOF

# /etc/containers/containers.conf — crun runtime, systemd cgroups, ulimits
test -f /etc/containers/containers.conf && \
  cp /etc/containers/containers.conf /etc/containers/containers.conf.orig

cat >/etc/containers/containers.conf <<'EOF'
[containers]
default_ulimits = [
  "nofile=65535:65535",
  "nproc=4096:4096",
]
log_size_max = 104857600
pids_limit = 4096

[engine]
cgroup_manager = "systemd"
runtime = "crun"
EOF

# Mask the rootful podman socket — Console never uses it
systemctl disable --now podman.socket 2>/dev/null || true
systemctl mask podman.socket
```

**Verify:**
```bash
podman info | grep -E 'overlay|seccomp|selinux|cgroupManager|runtime'
podman --version
```

## Phase 6 — TLS material `[A]`

```bash
mkdir -p /etc/pki/prisma
install -m 0644 /root/prisma/console.crt     /etc/pki/prisma/console.crt
install -m 0600 /root/prisma/console.key     /etc/pki/prisma/console.key
install -m 0644 /root/prisma/ca-chain.crt    /etc/pki/prisma/ca-chain.crt

# Verify key + cert match
openssl x509 -modulus -noout -in /etc/pki/prisma/console.crt | openssl md5
openssl rsa  -modulus -noout -in /etc/pki/prisma/console.key | openssl md5
# ^ both md5 sums must be identical
openssl verify -CAfile /etc/pki/prisma/ca-chain.crt /etc/pki/prisma/console.crt
```

## Phase 7 — Stage the installer `[A]`

```bash
mkdir -p /opt/prisma-install
cp /root/prisma/prisma_cloud_compute_edition_34_01_126.tar.gz        /opt/prisma-install/
cp /root/prisma/prisma_cloud_compute_edition_34_01_126.tar.gz.sha256 /opt/prisma-install/

cd /opt/prisma-install
sha256sum -c prisma_cloud_compute_edition_34_01_126.tar.gz.sha256   # "OK"
tar -xzf prisma_cloud_compute_edition_34_01_126.tar.gz              # extracts flat into /opt/prisma-install/

ls -la                                                              # expect twistlock.sh, linux/twistcli, twistlock.cfg*, images/
chmod 0750 twistlock.sh
```

## Phase 8 — Render `twistlock.cfg` `[A]`

Start from the shipped example (`twistlock.cfg.example` or similar),
fill in the non-secret values, **reconcile field names against the
shipped file if in doubt** ([runbook §2.4](../runbook.md#24-authoritative-sources) —
the shipped file wins over anything in the runbook or this doc).

Representative fields to set (air-gapped + FIPS):
- `DATA_FOLDER=/var/lib/twistlock`
- `MANAGEMENT_PORT_HTTPS=8083`
- `COMMUNICATION_PORT=8084`
- `FIPS_ENABLED=true`
- `CUSTOM_CONSOLE_CERT=/etc/pki/prisma/console.crt`
- `CUSTOM_CONSOLE_KEY=/etc/pki/prisma/console.key`
- `CUSTOM_CA_CHAIN=/etc/pki/prisma/ca-chain.crt`
- LDAPS binding fields — bind DN, bind password (from vault), search
  base, user filter, group filter, server cert path.

Place the rendered file at the location `twistlock.sh` expects
(inside the extracted dir, usually next to `twistlock.sh`).

## Phase 9 — Run the installer `[M]`

**This phase is manual by design** — `./twistlock.sh` is interactive
and compliance-load-bearing.

```bash
cd /opt/prisma-install/prisma_cloud_compute_edition_34_01_126
./twistlock.sh -s console 2>&1 | tee /root/twistlock-install.log
```

Answer prompts per the rendered `twistlock.cfg` (the installer will
often just confirm the fields already set). On success the installer
creates `/etc/systemd/system/twistlock.service` and starts the
Console container.

**Verify:**
```bash
systemctl status twistlock
podman ps                          # twistlock_console container, Up
curl -sk https://localhost:8083/api/v1/_ping
```

## Phase 10 — Operations hardening `[A]`

### 10.1 systemd override

```bash
mkdir -p /etc/systemd/system/twistlock.service.d
cat >/etc/systemd/system/twistlock.service.d/override.conf <<'EOF'
[Service]
Restart=on-failure
RestartSec=10s
StandardOutput=journal
StandardError=journal
CPUAccounting=yes
MemoryAccounting=yes
IOAccounting=yes
TasksMax=8192
EOF
systemctl daemon-reload
systemctl restart twistlock
```

### 10.2 logrotate

```bash
cat >/etc/logrotate.d/prisma <<'EOF'
/var/lib/twistlock/log/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    su root root
}
EOF
logrotate -d /etc/logrotate.d/prisma   # dry-run
```

### 10.3 Backup — nightly push to internal artefact store

Licence the service account in the Console UI: `Manage → Authentication
→ Users → Add user`, role = *CI Access User*, grab the auth token.

```bash
# Secrets go in /etc/default/prisma-backup.env (mode 0600, root-owned).
install -d -m 0700 -o root -g root /var/lib/twistlock-backup
cat >/etc/default/prisma-backup.env <<'EOF'
PRISMA_BACKUP_TOKEN=<from-vault>
INTERNAL_BACKUP_URL=https://artefacts.internal/prisma/
PRISMA_BACKUP_DIR=/var/lib/twistlock-backup    # NFS mount, shared primary ↔ secondary
PRISMA_BACKUP_RETENTION_DAYS=30
CONSOLE_URL=https://localhost:8083
EOF
chmod 0600 /etc/default/prisma-backup.env

# Install /usr/local/sbin/prisma-backup.sh — copy from the
# roles/prisma_backup/templates/prisma-backup.sh.j2 template, substituting
# the variables above, or see runbook §9 for the full script body.

# Systemd timer @ 02:15 UTC ±5m jitter.
cat >/etc/systemd/system/prisma-backup.service <<'EOF'
[Unit]
Description=Prisma Console nightly backup
After=twistlock.service
[Service]
Type=oneshot
EnvironmentFile=/etc/default/prisma-backup.env
ExecStart=/usr/local/sbin/prisma-backup.sh
EOF
cat >/etc/systemd/system/prisma-backup.timer <<'EOF'
[Unit]
Description=Run Prisma backup nightly
[Timer]
OnCalendar=*-*-* 02:15:00
RandomizedDelaySec=300
Persistent=true
[Install]
WantedBy=timers.target
EOF
systemctl daemon-reload
systemctl enable --now prisma-backup.timer
```

### 10.4 Monitoring — node_exporter + rsyslog → SIEM

Stage and run `node_exporter` (port 9100); open firewalld for 9100/tcp
from the Prometheus source CIDR. Configure `/etc/rsyslog.d/90-prisma.conf`
to forward to SIEM on 514/tcp. Full templates in
`roles/prisma_monitoring/`.

## Verification

```bash
# FIPS
cat /proc/sys/crypto/fips_enabled                     # 1
update-crypto-policies --show                         # FIPS

# Console
systemctl is-active twistlock
podman ps
curl -sk https://localhost:8083/api/v1/_ping          # HTTP 200

# TLS
echo | openssl s_client -connect localhost:8083 -servername prisma-console.internal 2>/dev/null | openssl x509 -noout -subject -issuer -dates

# Backup
systemctl list-timers prisma-backup.timer
ls -la /var/lib/twistlock-backup/                     # empty until first run

# Journald + rsyslog
journalctl -u twistlock -n 50 --no-pager
logger -t prisma-manual-test "hello SIEM"             # verify in SIEM

# Firewall
firewall-cmd --list-all                               # 8083, 8084, 22, 9100 (if prom)
```

## Troubleshooting

- **FIPS check fails after Phase 2 reboot.** Kernel failed to
  initialise FIPS — `dmesg | grep -i fips`. Usually a
  `/boot/grub2/grub.cfg` regeneration issue; rerun `fips-mode-setup
  --enable --no-bootcfg` and regenerate grub cfg manually.
- **`sha256sum -c` fails in Phase 7.** Either the tarball was
  truncated during staging or the `.sha256` file encodes a different
  filename. Recompute with `sha256sum <file>` and compare.
- **`twistlock.sh` exits non-zero in Phase 9.** The installer logs
  to `/var/lib/twistlock/log/install.log` in addition to stdout; the
  first ERROR line there tells you which field in `twistlock.cfg` it
  rejected.
- **Console UI won't load on 8083.** `podman ps` — is the container
  up? `podman logs twistlock_console | tail -50` — TLS cert path or
  LDAPS reachability is the usual suspect.
- **Backup timer doesn't fire.** `journalctl -u prisma-backup.service
  -n 50`. Curl PUT failures usually mean the `INTERNAL_BACKUP_URL`
  path doesn't exist; the script does not create intermediate
  directories on the remote side.

## Cross-references

- [Full runbook](../runbook.md) — acceptance-criteria tables,
  rollback procedures, §11 troubleshooting expanded, §12 config
  samples.
- [FIPS feasibility](../fips-feasibility.md) — Console FIPS posture
  in detail.
- [Ansible roles](../../roles/) — each role's `README.md` documents
  the variables it consumes. `tasks/main.yml` mirrors the commands
  above exactly.
