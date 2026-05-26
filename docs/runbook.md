# Prisma Cloud Compute Edition — Console Install Runbook
## Version 34.01.126 · RHEL 8.10 · podman · FIPS 140-2 · Air-gapped

---

### Document control

| Field | Value |
| --- | --- |
| Document | Prisma Cloud Compute Console — Airgapped Install Runbook (greenfield) |
| Target version | Prisma Cloud Compute Edition 34.01.126 |
| Target platform | RHEL 8.10 (Ootpa), rootful podman 4.4.1 |
| Compliance | FIPS 140-2 Level 1 (host kernel + OpenSSL + Console) |
| Deployment size | < 100 Defenders (lab / small production) |
| Air-gap | Yes — no outbound internet from Console VM |
| Automation | Ansible preferred for all repeatable, idempotent steps |
| Starting state | Fresh RHEL 8.10 VM + Prisma tarball + licence already in hand |
| Status | **Draft — pending review** |
| Owner | DevSecOps SME |

### Change log
- v0.1 — Initial draft (remediation).
- v0.2 — Rewritten as greenfield FIPS install.
- v0.3 — Enterprise-grade runbook.
- v0.4 — Added full dependency install, podman configuration, log rotation, and explicit Manual vs Ansible markers.

### Step classification legend
Each procedural step is tagged so the operator knows what to run by hand and what to drive from Ansible.

| Tag | Meaning |
| --- | --- |
| `[M]` **Manual** | Must be run by hand. Either interactive (first-run wizard), or a one-time decision with compliance weight (FIPS enable), or not safely repeatable. |
| `[A]` **Automate (Ansible)** | Idempotent, repeatable on every build. Runs from the control node. |
| `[A/M]` **Automate with manual verification** | Ansible does the work; operator must eyeball a result before proceeding. |

A full Ansible role outline covering every `[A]` step is in §14.

---

## 1. Purpose and scope

### 1.1 Purpose
Provide a step-by-step, copy-pasteable procedure that a single operator can follow — with Ansible automation for the repeatable parts — to install, configure, and validate a standalone Prisma Cloud Compute Console on a fresh RHEL 8.10 VM inside an air-gapped environment, with FIPS 140-2 mode enabled end-to-end (kernel → OpenSSL → container → Console) **from absolute zero** (dependency install, podman configuration, log rotation, systemd hardening, backup).

### 1.2 In scope
- Full OS baseline: FIPS kernel, SELinux, firewalld, chrony, sshd.
- Podman install and configuration: `registries.conf`, `storage.conf`, `containers.conf`, cgroup v2, journald limits.
- Storage: verify the DC-provided data volume is mounted; SELinux labelling.
- Prisma Console (onebox) install with `FIPS_ENABLED=true`, CA-signed TLS.
- systemd unit hardening, log rotation, nightly backup, monitoring exporter.
- Ansible role skeleton covering every automatable step.
- End-to-end verification including negative cipher-suite tests.

### 1.3 Out of scope
- Defender rollout.
- Highly-available / clustered Console.
- Data migration from the previous Console (handled separately if required; see §11.3).
- Kubernetes-based Console deployment.

---

## 2. Prerequisites and assumptions

### 2.1 Artefacts already in hand (operator has these)
- `prisma_cloud_compute_edition_34_01_126.tar.gz` — Prisma Compute installer (air-gapped).
- Prisma Cloud Compute Edition licence key.
- Internal CA-issued TLS certificate and key for the Console FQDN (plus CA chain).
- Internal LDAPS endpoint, bind DN, bind password, LDAPS-server CA cert.

### 2.2 Operator prerequisites
- Root SSH access to the target RHEL 8.10 VM.
- Sudo-capable Ansible control node on the same network segment, with SSH key auth to the target.
- Write access to an internal artefact store (for backups).
- Write access to an internal package mirror / Red Hat Satellite **or** a pre-populated dnf cache (the Console host itself does not reach the internet).

### 2.3 Environment assumptions
- Internal DNS resolves the Console FQDN (e.g. `prisma-console.<internal-domain>`).
- Internal NTP reachable (FIPS and Defender check-in rely on accurate time).
- Internal LDAPS reachable from the VM subnet.
- Internal SIEM and Prometheus reachable from the VM subnet.

### 2.4 Authoritative sources
The Prisma Cloud public documentation site (`docs.prismacloud.io`) is a single-page JavaScript app; several field specifics could not be retrieved. **The authoritative source for every field, flag, and path in this runbook is the files shipped inside the installer tarball:**

- `twistlock.cfg.example` — ground truth for configuration field names.
- `twistlock.sh` — ground truth for CLI flags and installer behaviour.
- The systemd unit shipped alongside (filename TBD) — ground truth for service semantics.
- `images/` directory — container image tarball(s).

Where this runbook specifies a field name, reconcile against the shipped file in Phase 4; if they differ, the **shipped file wins**.

---

## 3. Architecture overview

### 3.1 Target-state configuration

| Component | Specification |
| --- | --- |
| Hypervisor | VMware vSphere, CPU + memory hot-add enabled |
| OS | Red Hat Enterprise Linux 8.10 (Ootpa), minimal server profile |
| vCPU | 8 |
| RAM | 16 GiB |
| OS disk | 80 GiB (thick provisioned, local-SSD datastore) |
| Data disk | 500 GiB (thick provisioned, **local SSD — not NFS**) |
| Filesystem | XFS on LVM, `noatime`, `inode64` |
| SELinux | Enforcing, targeted policy |
| Kernel FIPS | `fips=1`, `fips-mode-setup` enabled |
| Cgroups | v2 (default on RHEL 8.10) |
| Container runtime | podman 4.4.1 (rootful), overlayfs graph, `crun` runtime |
| Prisma Console | v34.01.126, `FIPS_ENABLED=true` |
| Console ports | 8083/tcp (HTTPS UI + API), 8084/tcp (Defender websocket) |
| TLS | CA-signed, RSA 2048+ or ECDSA P-256+, SHA-256+ |
| Auth | LDAPS |
| Data directory | `/var/lib/twistlock` on the 500 GiB LV |
| Logs | journald + `/var/lib/twistlock/log/`, both under logrotate + SIEM |
| Restart policy | systemd `Restart=on-failure`, `RestartSec=10s` |
| Backup | Nightly `POST /api/v1/settings/backups` → internal artefact store; 30 d on-box retention |

### 3.2 Network flows

| Direction | Source | Destination | Port | Purpose |
| --- | --- | --- | --- | --- |
| In | Defender hosts | Console VM | 8084/tcp | Defender ↔ Console control channel |
| In | Operator browsers | Console VM | 8083/tcp | UI + API |
| In | Bastion | Console VM | 22/tcp | Admin SSH |
| Out | Console VM | Internal LDAPS | 636/tcp | Authentication |
| Out | Console VM | Internal NTP | 123/udp | Time sync |
| Out | Console VM | Internal DNS | 53/udp, 53/tcp | Resolution |
| Out | Console VM | Internal Satellite / mirror | 443/tcp | OS package updates |
| Out | Console VM | Internal artefact store | 443/tcp | Nightly backup shipping |
| Out | Console VM | Internal Prometheus / SIEM | 9100/tcp, 514/tcp | Metrics + log shipping |

---

## 4. Bill of materials

Confirm all artefacts are present and integrity-checked **before** starting Phase 5.

| # | Artefact | Location | Verification |
| --- | --- | --- | --- |
| 1 | `prisma_cloud_compute_edition_34_01_126.tar.gz` | operator laptop / artefact store | SHA-256 recorded from Palo Alto CSP |
| 2 | Prisma Compute Edition licence key | operator vault | non-expired, matching deployment |
| 3 | `console.crt`, `console.key`, `ca-chain.crt` | internal CA output | modulus match, chain complete |
| 4 | LDAPS server CA cert | internal CA / LDAPS admin | verifies `ldaps://` handshake |
| 5 | RHEL 8.10 ISO or kickstart | internal Satellite | Red Hat SHA-256 |
| 6 | Ansible control node | operator workstation or bastion | Ansible 2.15+, SSH key to target |
| 7 | This runbook | version-controlled repo | — |

---

## 5. Installation procedure

Execute Phases 1–10 in order. Each phase has explicit acceptance criteria. Do not proceed until the current phase's acceptance criteria pass.

**Conventions**
- Shell blocks marked with `# [M]` are intended for manual execution on the Console VM as root.
- Shell blocks marked with `# [A]` are the *equivalent* commands that the Ansible role (§14) performs — reproduce them in a module rather than running them raw.

---

### Phase 1 — VM provisioning (on vSphere) `[M]`

**Objective:** produce a fresh RHEL 8.10 VM sized per §3.1.

This phase is manual — it depends on vSphere clicks or a Terraform workflow that varies per site. Codify in Terraform if possible; otherwise document the VM spec in the change ticket.

**Steps**
1. Create a new VM:
   - Guest OS: Linux, Red Hat Enterprise Linux 8 (64-bit).
   - CPU: 8 vCPU, 4 cores/socket (2 sockets), CPU hot-add on.
   - Memory: 16384 MiB, memory hot-add on.
   - Disk 1 (OS): 80 GiB, thick, eager-zeroed, local SSD.
   - Disk 2 (data): 500 GiB, thick, eager-zeroed, local SSD.
   - NIC on the security-tools VLAN.
2. Boot RHEL 8.10 minimal-server install, use only Disk 1 for the OS volume group. The platform/DC team provisions Disk 2 as the Console data volume and mounts it at `/var/lib/twistlock` (XFS, not NFS) before handover.
3. Set hostname to the planned Console FQDN.
4. Register to internal Satellite (or configure offline dnf repo), `dnf -y update`, reboot.

**Acceptance criteria (Phase 1)**
- `hostnamectl` returns the correct FQDN.
- `lsblk` shows `sda` (80 GiB, used) and the 500 GiB data volume mounted at `/var/lib/twistlock`; `findmnt /var/lib/twistlock` confirms XFS, not NFS.
- `cat /etc/redhat-release` == `Red Hat Enterprise Linux release 8.10 (Ootpa)`.
- SSH key for the Ansible control node is installed at `/root/.ssh/authorized_keys` (or a dedicated sudo user).

---

### Phase 2 — Kernel FIPS enablement `[A/M]`

**Objective:** put the kernel + system crypto policy into FIPS mode.

This is `[A/M]`: Ansible runs the enable command; the **reboot and verification are always manually reviewed** because FIPS is compliance-load-bearing.

**Automated (Ansible)**
```yaml
- name: Enable FIPS mode
  ansible.builtin.command: fips-mode-setup --enable
  register: fips_enable
  changed_when: "'FIPS mode will be enabled' in fips_enable.stdout"

- name: Reboot to activate FIPS
  ansible.builtin.reboot:
    reboot_timeout: 600
  when: fips_enable.changed
```

**Manual verification after reboot**
```bash
# [M]
fips-mode-setup --check           # FIPS mode is enabled.
cat /proc/sys/crypto/fips_enabled # 1
update-crypto-policies --show     # FIPS
```

**Acceptance criteria (Phase 2)**
- All three outputs exactly as above.
- `dmesg | grep -i fips` shows the kernel FIPS module initialised without errors.

---

### Phase 3 — Host hardening baseline `[A]`

**Objective:** bring the VM to a hardened baseline under SELinux, firewalld, and chrony. Entirely automatable.

**What Ansible does** (role: `rhel_baseline`)
1. `SELinux=enforcing` in `/etc/selinux/config`; reboot if the live mode was not enforcing.
2. Install the OS dependency set:
   ```yaml
   ansible.builtin.dnf:
     name:
       - podman
       - lvm2
       - xfsprogs
       - chrony
       - firewalld
       - policycoreutils-python-utils
       - openssl
       - jq
       - tar
       - gzip
       - logrotate
       - rsyslog
       - audit
     state: present
   ```
3. Enable and start `chronyd`, `firewalld`, `auditd`, `rsyslog`.
4. Render `/etc/chrony.conf` pointing at internal NTP sources.
5. Harden `/etc/ssh/sshd_config`: `PermitRootLogin no`, `PasswordAuthentication no`, `PubkeyAuthentication yes`. Reload sshd.
   - **Admin password exception (opt-in, off by default):** these hosts otherwise
     accept only `publickey`/`gssapi`, so admin tooling that uses AD
     username+password (e.g. MobaXterm) cannot connect. Set
     `rhel_baseline_password_auth_exception: true` + `rhel_baseline_password_auth_cidr`
     (admin/jump CIDR) to drop a `Match Address` block in
     `/etc/ssh/sshd_config.d/50-prisma-admin-password.conf` that re-enables password
     auth **only** from that network. The global `no` is unchanged for everyone else.
     This loosens the baseline — scope the CIDR tightly; prefer keys/Kerberos where feasible.
6. Configure firewalld ports:
   ```yaml
   - { port: 8083/tcp, state: enabled }
   - { port: 8084/tcp, state: enabled }
   - { service: ssh,  state: enabled }
   ```
   Reload firewall.
7. Set a modest sysctl baseline (file `/etc/sysctl.d/90-prisma.conf`):
   ```
   vm.swappiness = 10
   vm.max_map_count = 262144
   fs.file-max = 2097152
   net.core.somaxconn = 1024
   net.ipv4.tcp_max_syn_backlog = 1024
   ```
8. Set journald limits (`/etc/systemd/journald.conf.d/10-prisma.conf`) to cap journal growth:
   ```
   [Journal]
   Storage=persistent
   SystemMaxUse=2G
   SystemKeepFree=1G
   MaxFileSec=1week
   Compress=yes
   ForwardToSyslog=yes
   ```
   Restart `systemd-journald`.

**Manual verification**
```bash
# [M]
getenforce                           # Enforcing
systemctl is-active chronyd          # active
chronyc tracking | head
firewall-cmd --list-all
sysctl vm.swappiness vm.max_map_count
journalctl --disk-usage
```

**Acceptance criteria (Phase 3)**
- All services `active`, firewall lists the expected ports, NTP offset small, sysctl values applied, journal disk usage bounded.

---

### Phase 4 — Storage: ensure the data folder + SELinux label `[A]`

**Objective:** ensure `/var/lib/twistlock` exists and is labelled for podman.
The role does **not** create or require a dedicated mount — whether the data
folder is its own filesystem is the platform/DC team's choice. If a dedicated
data volume exists, mount it at `/var/lib/twistlock` before this runs (see
Phase 1); otherwise the role just creates the directory on the existing fs.

**What Ansible does** (role: `prisma_storage`)
```yaml
# GUARD: refuse to run on an unsafe prisma_data_folder (/, /var, /etc, …) so a
# misconfigured value can never stamp container_file_t across the system.
- name: Validate prisma_data_folder is a safe, dedicated path
  ansible.builtin.assert:
    that:
      - (prisma_data_folder | default('')).startswith('/')
      - (prisma_data_folder | default('')) not in prisma_storage_unsafe_paths
      - ((prisma_data_folder | default('')) | regex_replace('/+$','')).split('/') | length >= 3

- name: Ensure the data folder exists
  ansible.builtin.file:
    path: /var/lib/twistlock
    state: directory
    owner: root
    group: root
    mode: "0750"

# SELinux fcontext via the semanage CLI (community.general not available in the
# air-gapped EE). semanage from policycoreutils-python-utils (rhel_baseline).
- name: SELinux fcontext for data dir
  ansible.builtin.command: semanage fcontext -a -t container_file_t '/var/lib/twistlock(/.*)?'
  # idempotent: tolerate "already defined"

- name: Relabel
  ansible.builtin.command: restorecon -R /var/lib/twistlock
  changed_when: false
```

**Manual verification**
```bash
# [M]
ls -Zd /var/lib/twistlock     # context shows container_file_t
df -hT /var/lib/twistlock     # confirm capacity (dedicated volume optional)
```

**Acceptance criteria (Phase 4)**
- `/var/lib/twistlock` exists and is labelled `container_file_t` (NOT NFS).
  A dedicated mount is recommended for capacity/IO isolation but not required
  or asserted. `prisma_data_folder` must be a safe dedicated path — the role
  refuses to label a system directory.

---

### Phase 5 — Podman installation and configuration `[A]`

**Objective:** make podman production-ready under FIPS and SELinux for a rootful Console container.

Podman itself was installed in Phase 3. This phase is pure **configuration**.

#### 5.1 Confirm runtime `[A/M]`
```bash
# [M]
podman --version                                   # 4.4.1 or newer
podman info | grep -E 'graphDriverName|runtime|SecurityOptions|cgroupVersion'
```
Expect: `graphDriverName: overlay`, `cgroupVersion: v2`, `runtimePath: crun` (or `runc`), `SecurityOptions: seccomp selinux`.

If podman is running rootless by accident, switch to rootful; the Console must run under `root`.

#### 5.2 `/etc/containers/registries.conf` `[A]`
Airgapped — block unintended pulls. Render via Ansible template.
```toml
unqualified-search-registries = []

[[registry]]
location = "registry.local"
insecure = false
blocked = false

[[registry]]
location = "docker.io"
blocked = true

[[registry]]
location = "quay.io"
blocked = true
```
Only allow an internal registry you actually operate. If there is no internal registry, block everything and rely solely on the tarball.

#### 5.3 `/etc/containers/storage.conf` `[A]`
```toml
[storage]
driver = "overlay"
runroot = "/run/containers/storage"
graphroot = "/var/lib/containers/storage"

[storage.options]
additionalimagestores = []

[storage.options.overlay]
mountopt = "nodev,metacopy=on"
```
Keep the podman graph on the **OS disk** (`/var/lib/containers`), and keep only the Prisma data bind-mount on the dedicated LV. This isolates Console data from podman image churn.

#### 5.4 `/etc/containers/containers.conf` `[A]`
```toml
[containers]
log_driver = "journald"
log_size_max = 104857600        # 100 MiB per container log, then journald rotates
pids_limit = 4096
default_ulimits = [
  "nofile=65535:65535",
  "nproc=4096:4096",
]

[engine]
cgroup_manager = "systemd"
events_logger = "journald"
runtime = "crun"
```
Using `cgroup_manager = "systemd"` integrates cgroup v2 accounting with systemd — metrics are then visible via `systemd-cgtop` and scrapable by cAdvisor / node_exporter cgroup collectors.

#### 5.5 Enable (do not enable) the podman socket `[A]`
```yaml
# Leave disabled — the Console does not need the podman API socket.
- name: Mask podman socket
  ansible.builtin.systemd:
    name: podman.socket
    enabled: false
    masked: true
```

#### 5.6 Reload and smoke test `[A/M]`
```bash
# [M]
systemctl daemon-reload
podman info | grep -E 'cgroupManager|logDriver|eventLogger'
# Expect: cgroupManager: systemd, logDriver: journald, eventLogger: journald
```

**Acceptance criteria (Phase 5)**
- `podman info` reports `graphDriverName: overlay`, `cgroupVersion: v2`, `cgroupManager: systemd`, `logDriver: journald`, `SecurityOptions: seccomp selinux`.
- `/etc/containers/registries.conf` blocks all unintended external registries.
- `podman.socket` is masked.

---

### Phase 6 — Stage TLS material `[A]`

**Objective:** land cert, key, and CA chain with the correct perms and SELinux labels.

```yaml
- name: Create pki dir
  ansible.builtin.file:
    path: /etc/pki/prisma
    state: directory
    owner: root
    group: root
    mode: "0750"

- name: Place cert/key/chain
  ansible.builtin.copy:
    src: "{{ item.src }}"
    dest: "/etc/pki/prisma/{{ item.dest }}"
    owner: root
    group: root
    mode: "{{ item.mode }}"
  loop:
    - { src: console.crt,   dest: console.crt,   mode: "0600" }
    - { src: console.key,   dest: console.key,   mode: "0600" }
    - { src: ca-chain.crt,  dest: ca-chain.crt,  mode: "0644" }

- name: Restore SELinux context
  ansible.builtin.command: restorecon -R /etc/pki/prisma
  changed_when: false
```

**Manual verification**
```bash
# [M]
openssl x509 -noout -modulus -in /etc/pki/prisma/console.crt | openssl sha256
openssl rsa  -noout -modulus -in /etc/pki/prisma/console.key | openssl sha256
# Both hashes must match.
openssl verify -CAfile /etc/pki/prisma/ca-chain.crt /etc/pki/prisma/console.crt
# Expect: OK
```

**Acceptance criteria (Phase 6)**
- Cert/key modulus hashes match.
- `openssl verify` returns `OK`.

---

### Phase 7 — Stage and extract the Prisma installer `[A/M]`

**Objective:** land the (already-downloaded) Prisma tarball on the VM and extract it in a known location.

**Automated**
```yaml
- name: Ensure install dir
  ansible.builtin.file:
    path: /opt/prisma-install
    state: directory
    owner: root
    group: root
    mode: "0750"

- name: Copy Prisma tarball
  ansible.builtin.copy:
    src: prisma_cloud_compute_edition_34_01_126.tar.gz
    dest: /opt/prisma-install/prisma_cloud_compute_edition_34_01_126.tar.gz
    owner: root
    group: root
    mode: "0600"

- name: Verify SHA-256
  ansible.builtin.command: sha256sum -c prisma_cloud_compute_edition_34_01_126.tar.gz.sha256
  args:
    chdir: /opt/prisma-install

- name: Extract installer
  ansible.builtin.unarchive:
    src: /opt/prisma-install/prisma_cloud_compute_edition_34_01_126.tar.gz
    dest: /opt/prisma-install/
    remote_src: yes
    creates: /opt/prisma-install/prisma_cloud_compute_edition_34_01_126/twistlock.sh
```

**Manual step — reconcile shipped field names** `[M]`
```bash
cd /opt/prisma-install/prisma_cloud_compute_edition_34_01_126
ls -la
grep -nE 'FIPS|DATA_FOLDER|MANAGEMENT_PORT|CUSTOM_CERT|CGROUP|MEMORY' twistlock.cfg*
grep -nE 'fips|FIPS|--memory|systemctl|-s console' twistlock.sh
ls images/ 2>/dev/null || true
```
Record the **exact** field names used in the shipped `twistlock.cfg.example`. If any name differs from §12.1, update §12.1 in the controlled copy of this runbook before proceeding.

**Acceptance criteria (Phase 7)**
- SHA-256 matches the value recorded from the Palo Alto CSP when the tarball was originally downloaded.
- Installer directory contains `twistlock.sh` (executable), a config file, and an `images/` tarball (or equivalent bundled image).
- Operator has recorded the shipped field names.

---

### Phase 8 — Render and review `twistlock.cfg` `[A/M]`

**Objective:** produce the final configuration the installer will consume.

**Automated (template)**
Ansible renders `/opt/prisma-install/prisma_cloud_compute_edition_34_01_126/twistlock.cfg` from a Jinja template (see §12.1) using variables:
```yaml
prisma_fips_enabled: true
prisma_data_folder: /var/lib/twistlock
prisma_https_port: 8083
prisma_comm_port: 8084
prisma_custom_cert: true
prisma_cert_path: /etc/pki/prisma/console.crt
prisma_cert_key_path: /etc/pki/prisma/console.key
prisma_cert_ca_path: /etc/pki/prisma/ca-chain.crt
```
Before rendering, Ansible keeps a copy of the original at `twistlock.cfg.orig` for rollback.

**Manual review** `[M]`
```bash
cd /opt/prisma-install/prisma_cloud_compute_edition_34_01_126
diff twistlock.cfg.orig twistlock.cfg
grep -E '^(FIPS_ENABLED|DATA_FOLDER|MANAGEMENT_PORT_HTTPS|COMMUNICATION_PORT|CUSTOM_CERT)' twistlock.cfg
```

**Acceptance criteria (Phase 8)**
- `twistlock.cfg` contains FIPS + TLS + data-folder values.
- `twistlock.cfg.orig` exists.

---

### Phase 9 — Run the installer `[M]`

**Objective:** run `twistlock.sh`. This is `[M]` — Palo Alto's installer is interactive in places, may prompt on cert trust, and you want to see its output live rather than through an Ansible wrapper.

```bash
# [M]
cd /opt/prisma-install/prisma_cloud_compute_edition_34_01_126
./twistlock.sh -s console | tee /root/twistlock-install.log
```

The installer typically:
- Loads the Console image into podman (`podman load -i images/...`).
- Renders a systemd unit (e.g. `twistlock.service`) into `/etc/systemd/system/`.
- Creates the data-folder layout under `/var/lib/twistlock`.
- Starts the Console container.

Watch the log. If it fails on image checksum or podman detection, stop and diagnose — do not retry with a corrupt tarball.

**Smoke test** `[M]`
```bash
systemctl status twistlock
podman ps
ss -lntp | grep -E '8083|8084'
podman logs twistlock_console 2>&1 | grep -i fips
```

**Acceptance criteria (Phase 9)**
- `systemctl is-active twistlock` == `active`.
- `podman ps` shows Console container `Up`.
- Both `:8083` and `:8084` bound.
- Console log contains a FIPS banner line.

---

### Phase 10 — systemd, log rotation, backup, monitoring `[A]`

**Objective:** everything the shipped installer does not ship. Entirely automatable.

#### 10.1 systemd hardening `[A]`
Drop-in at `/etc/systemd/system/twistlock.service.d/override.conf`:
```
[Service]
Restart=on-failure
RestartSec=10s
# Limit journald noise:
StandardOutput=journal
StandardError=journal
# Resource accounting (cgroup v2 via systemd):
CPUAccounting=yes
MemoryAccounting=yes
IOAccounting=yes
TasksMax=8192
```
```yaml
- name: systemd override
  ansible.builtin.copy:
    dest: /etc/systemd/system/twistlock.service.d/override.conf
    content: "{{ twistlock_override }}"
  notify: daemon-reload-and-restart

- name: Enable + persist
  ansible.builtin.systemd:
    name: twistlock
    enabled: true
```

#### 10.2 Log rotation `[A]`

The Console writes application logs under `/var/lib/twistlock/log/` (mounted through to the container). journald handles container stdout. Both need bounds.

**File:** `/etc/logrotate.d/prisma-console`
```
/var/lib/twistlock/log/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    dateext
    dateformat -%Y%m%d
    create 0640 root root
    sharedscripts
}
```
Using `copytruncate` avoids having to signal the running container to reopen log files.

```yaml
- name: Install logrotate policy
  ansible.builtin.copy:
    src: prisma-console.logrotate
    dest: /etc/logrotate.d/prisma-console
    owner: root
    group: root
    mode: "0644"

- name: Smoke test logrotate
  ansible.builtin.command: logrotate -d /etc/logrotate.d/prisma-console
  changed_when: false
```

journald is already capped via `/etc/systemd/journald.conf.d/10-prisma.conf` in Phase 3.

#### 10.3 Nightly backup `[A]`

**Script** `/usr/local/sbin/prisma-backup.sh` (mode `0750`, owner `root`):
```bash
#!/usr/bin/env bash
set -euo pipefail
ts=$(date -u +%Y%m%dT%H%M%SZ)
out="/var/lib/twistlock-backup/prisma-${ts}.tar.gz"
mkdir -p /var/lib/twistlock-backup
curl -ks --fail \
  -u "${PRISMA_BACKUP_USER}:${PRISMA_BACKUP_TOKEN}" \
  -o "${out}" \
  "https://localhost:8083/api/v1/backups"
# Ship to internal artefact store
curl -ks --fail -T "${out}" "${INTERNAL_BACKUP_URL}/$(basename "${out}")"
# On-box retention 30 d
find /var/lib/twistlock-backup -type f -mtime +30 -delete
```

**Credentials**: `/etc/prisma/backup.env` (mode `0600`, owner `root`):
```
PRISMA_BACKUP_USER=<service-account>
PRISMA_BACKUP_TOKEN=<token-from-vault>
INTERNAL_BACKUP_URL=https://artefacts.<internal-domain>/prisma/
```

**systemd unit** `/etc/systemd/system/prisma-backup.service`:
```
[Unit]
Description=Nightly Prisma Console backup
After=network-online.target twistlock.service
Requires=twistlock.service

[Service]
Type=oneshot
EnvironmentFile=/etc/prisma/backup.env
ExecStart=/usr/local/sbin/prisma-backup.sh
User=root
```

**systemd timer** `/etc/systemd/system/prisma-backup.timer`:
```
[Unit]
Description=Run Prisma backup nightly

[Timer]
OnCalendar=*-*-* 02:15:00
Persistent=true
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

```yaml
- name: Enable backup timer
  ansible.builtin.systemd:
    name: prisma-backup.timer
    enabled: true
    state: started
    daemon_reload: true
```

#### 10.4 Monitoring `[A]`
1. Install `node_exporter` and enable its systemd unit. Scrape from internal Prometheus.
2. Alert rules:
   - Container RSS > 80 % of cgroup cap for 10 min.
   - `/api/v1/_ping` > 2 s for 5 min (via blackbox_exporter).
   - `/var/lib/twistlock` filesystem growth > 10 %/week.
   - `twistlock.service` not `active`.

#### 10.5 Log shipping `[A]`
`rsyslog` imfile to forward `/var/lib/twistlock/log/*.log` to the internal SIEM; journald forwarding already enabled via Phase 3.

**Acceptance criteria (Phase 10)**
- `systemctl is-enabled twistlock prisma-backup.timer` both `enabled`.
- `logrotate -d /etc/logrotate.d/prisma-console` reports the expected rotations without errors.
- A manual run of `prisma-backup.sh` produces a tarball in `/var/lib/twistlock-backup/` and lands a copy at the internal artefact store.
- Prometheus scrape target for this host is `UP`.
- SIEM receives Console journal events within 5 min.

---

### Phase 11 — First-run wizard, licence, auth `[M]`

This is `[M]` — interactive.

1. Browse to `https://prisma-console.<internal-domain>:8083/`. No browser cert warning should appear; if any does, stop and fix CA-chain distribution rather than bypassing.
2. Create the initial admin; store credentials in the enterprise vault.
3. Upload the licence key.
4. **Manage → Authentication → LDAP**: configure `ldaps://<internal-ldap>:636`, bind DN, bind password, upload LDAPS CA if not already trusted, map directory groups to Prisma roles per the RBAC matrix. Test bind. Log out, log in as a directory user.
5. **Manage → System**: set retention explicitly — scan reports 30 d, audits 90 d, vulnerability data 30 d. Record the chosen values in the change ticket.
6. **Alerts**: at minimum enable Critical/High vulnerabilities with a fix available and compliance failures on production-tagged images. Wire to SIEM / email alias.

**Acceptance criteria (Phase 11)**
- Valid licence shown under **Manage → System → License**.
- Directory-sourced user can log in with correct role.
- Retention explicitly set; alert rules active.

---

## 6. End-to-end verification `[A/M]`

Run after Phase 10 + Phase 11.

### 6.1 FIPS is actually active
```bash
# [M]
cat /proc/sys/crypto/fips_enabled   # 1
update-crypto-policies --show       # FIPS
podman logs twistlock_console 2>&1 | grep -i fips

# Negative — weak cipher must be refused
openssl s_client -connect prisma-console.<internal-domain>:8083 \
  -cipher 'RC4-SHA' </dev/null 2>&1 | grep -E 'handshake failure|no cipher'

# Positive — AES-GCM / ECDHE must be offered
openssl s_client -connect prisma-console.<internal-domain>:8083 \
  -tls1_2 </dev/null 2>&1 | grep -E 'Cipher\s+:\s+(ECDHE|AES(128|256)-GCM)'
```

### 6.2 Responsiveness
```bash
curl -ks -o /dev/null -w 'ping=%{time_total}s http=%{http_code}\n' \
  https://prisma-console.<internal-domain>:8083/api/v1/_ping
# Expect http=200, time_total < 0.5 s.
```
Browser dashboard < 10 s from a cold session.

### 6.3 Headroom
```bash
podman stats --no-stream twistlock_console
free -h
df -hT /var/lib/twistlock
```
RSS 1–2 GiB; no swap in use; data dir < 5 GiB on day one.

### 6.4 Auth
LDAPS login works, role mapping matches RBAC matrix.

### 6.5 Backup / restore drill
Trigger `prisma-backup.service` manually, confirm the tarball lands in `/var/lib/twistlock-backup/` and at the internal artefact store. Perform a restore drill in a lab VM (§11.3).

### 6.6 Sign-off
Capture all §6.1–§6.5 outputs in the change ticket; obtain DevSecOps SME + Security Engineering Lead signatures before registering Defenders.

---

## 7. Rollback and recovery

### 7.1 Rollback of an in-flight install
| Stage of failure | Action |
| --- | --- |
| Phase 1–4 fails | Re-deploy the VM from scratch (or re-run Ansible — the roles are idempotent). |
| Phase 5 (podman) fails | Reset podman config (restore shipped `registries.conf` / `storage.conf` / `containers.conf` backups Ansible keeps). |
| Phase 6 fails | Re-stage cert material. |
| Phase 7 fails (bad tarball) | Delete, re-transfer, re-verify SHA-256. |
| Phase 8 fails | Restore `twistlock.cfg.orig`. |
| Phase 9 fails | Stop and remove the Console container and systemd unit, clear `/var/lib/twistlock/*`, re-run from Phase 8 after correcting cause. |
| Phase 10–11 fails | Preserve the VM, fix forward. |

### 7.2 Uninstall

Quick reference below. For a full host teardown (services, podman, systemd units, config, optional LVM/firewall) and a clean re-install, see [`docs/uninstall-prisma-console.md`](uninstall-prisma-console.md).

```bash
# [M]
systemctl stop twistlock prisma-backup.timer
systemctl disable twistlock prisma-backup.timer
podman rm -f twistlock_console
podman rmi $(podman images -q --filter reference='*twistlock*')
rm -f /etc/systemd/system/twistlock.service
rm -rf /etc/systemd/system/twistlock.service.d
rm -f /etc/systemd/system/prisma-backup.{service,timer}
systemctl daemon-reload
# /var/lib/twistlock retained unless the VM is being decommissioned.
```

---

## 8. Operational runbook

### 8.1 Start / stop / restart
```bash
systemctl start twistlock
systemctl stop twistlock
systemctl restart twistlock      # graceful
systemctl status twistlock
```

### 8.2 Logs
- systemd journal: `journalctl -u twistlock -f`
- Container stdout: `podman logs -f twistlock_console`
- Application logs: `/var/lib/twistlock/log/`
- UI download: **Manage → Logs → Console**

### 8.3 Health checks
- `curl -ks https://localhost:8083/api/v1/_ping`
- `podman stats --no-stream twistlock_console`
- `df -hT /var/lib/twistlock`
- `systemd-cgtop | head`

### 8.4 Minor version upgrades
1. Land the new tarball at `/opt/prisma-install/`, verify SHA-256.
2. Trigger a backup (§9.1).
3. Extract into a new directory.
4. Copy current `twistlock.cfg` forward; reconcile any new required fields against the new `twistlock.cfg.example`.
5. Run the installer; it stops, replaces, and restarts in place, preserving `/var/lib/twistlock`.
6. Re-run §6 verification.

---

## 9. Backup and restore

### 9.1 On-demand backup
```bash
systemctl start prisma-backup.service
# or:
/usr/local/sbin/prisma-backup.sh
```

### 9.2 Scheduled backup
`prisma-backup.timer` fires nightly at 02:15 local (+ random 0–5 min jitter), 30 d on-box retention, copies off-box to the internal artefact store.

### 9.3 Restore
1. Provision a new VM through Phases 1–10 to the **same Console version**.
2. `systemctl stop twistlock`.
3. Upload backup tarball via UI **Manage → Utilities → Restore** or the `POST /api/v1/backups/{id}/restore` API (follow Palo Alto-documented v34 restore flow).
4. `systemctl start twistlock`.
5. Re-run §6 before pointing Defenders at the restored Console.

Run a restore drill quarterly.

---

## 10. Acceptance sign-off

- [ ] Phases 1–11 acceptance criteria green.
- [ ] §6.1 FIPS kernel, crypto policy, container banner, weak-cipher refusal, AES-GCM offered.
- [ ] §6.2 responsiveness: `/api/v1/_ping` < 500 ms, dashboard < 10 s.
- [ ] §6.3 headroom: RSS 1–2 GiB, no swap.
- [ ] §6.4 LDAPS login with correct role.
- [ ] §6.5 backup + restore drill completed.
- [ ] Runbook §8 walked through with the on-call team.

Sign-off: DevSecOps SME + Security Engineering Lead (dual approval).

---

## 11. Troubleshooting

| Symptom | Likely cause | First action |
| --- | --- | --- |
| `twistlock.sh` fails with "container runtime not found" | podman not installed or masked | `podman info`; re-check Phase 5 |
| Installer image checksum fails | Tarball corrupted in transfer | Re-transfer, re-verify SHA-256; do not retry with corrupt tarball |
| Console starts but UI shows cert error | Cert/key mismatch or incomplete CA chain | §Phase 6 verification |
| `podman logs` shows `fips` errors on startup | Host kernel not in FIPS | `cat /proc/sys/crypto/fips_enabled` must be `1`; re-run Phase 2 |
| `/api/v1/_ping` > 2 s | DB bloat or disk-I/O starvation | `du -sh /var/lib/twistlock`; confirm data disk on local SSD |
| LDAPS bind fails under FIPS | LDAPS cert weak signature or non-FIPS cipher | Upgrade LDAPS server cert to RSA-SHA256+ |
| Container exit 137 (OOM) | Retention not set or runaway integration | §Phase 11 step 5 retention; review **Monitor → Scans** |
| SELinux AVC denials in `audit.log` | Missing `container_file_t` on `/var/lib/twistlock` | Re-run Phase 4 SELinux task |
| Journal on root fills | journald limits not applied | Verify Phase 3 step 8 config |
| Container logs not rotating | logrotate not run or path mismatch | `logrotate -d /etc/logrotate.d/prisma-console` |

---

## 12. Appendices — config samples

### 12.1 `twistlock.cfg` template (Jinja)
Reconcile field names against the shipped `twistlock.cfg.example` before first use.
```jinja
FIPS_ENABLED={{ prisma_fips_enabled | string | lower }}
DATA_FOLDER={{ prisma_data_folder }}
MANAGEMENT_PORT_HTTPS={{ prisma_https_port }}
COMMUNICATION_PORT={{ prisma_comm_port }}
CUSTOM_CERT={{ prisma_custom_cert | string | lower }}
CUSTOM_CERT_PATH={{ prisma_cert_path }}
CUSTOM_CERT_KEY_PATH={{ prisma_cert_key_path }}
CUSTOM_CERT_CA_PATH={{ prisma_cert_ca_path }}
```

### 12.2 systemd override
`/etc/systemd/system/twistlock.service.d/override.conf`:
```
[Service]
Restart=on-failure
RestartSec=10s
CPUAccounting=yes
MemoryAccounting=yes
IOAccounting=yes
TasksMax=8192
```

### 12.3 logrotate policy
`/etc/logrotate.d/prisma-console`:
```
/var/lib/twistlock/log/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    copytruncate
    dateext
    dateformat -%Y%m%d
    create 0640 root root
    sharedscripts
}
```

### 12.4 journald caps
`/etc/systemd/journald.conf.d/10-prisma.conf`:
```
[Journal]
Storage=persistent
SystemMaxUse=2G
SystemKeepFree=1G
MaxFileSec=1week
Compress=yes
ForwardToSyslog=yes
```

### 12.5 Firewall summary
| Port | Proto | Direction | Source | Purpose |
| --- | --- | --- | --- | --- |
| 8083 | tcp | in | operator subnet | UI + API |
| 8084 | tcp | in | defender subnets | Defender control channel |
| 22 | tcp | in | bastion | Admin SSH |
| 636 | tcp | out | — | LDAPS |
| 123 | udp | out | — | NTP |
| 53 | udp/tcp | out | — | DNS |
| 443 | tcp | out | Satellite, backup store | Updates + backup shipping |

### 12.6 Known documentation gaps (v34 public docs portal — SPA-rendered)
Must be confirmed against shipped tarball files during Phase 7:
- Exact field name for FIPS (this runbook uses `FIPS_ENABLED`).
- Exact filename of the shipped systemd unit.
- Whether `twistlock.sh` auto-loads images for every rootful-podman variant.
- Enumerated list of disabled cipher suites under FIPS.
- Presence or absence of a `/api/v1/settings/fips` endpoint.

---

## 13. Manual vs Ansible split (summary)

| Phase | Work | `[M]` or `[A]` |
| --- | --- | --- |
| 1. VM provisioning | vSphere / Terraform | `[M]` (or site-specific automation) |
| 2. FIPS kernel enable | `fips-mode-setup --enable` + reboot | `[A/M]` |
| 3. Host hardening | packages, SELinux, firewalld, chrony, sshd, sysctl, journald | `[A]` |
| 4. Storage | verify data volume mounted (XFS) + SELinux label | `[A]` |
| 5. Podman config | registries/storage/containers.conf, socket masked | `[A]` |
| 6. TLS staging | copy cert/key/CA, perms, SELinux | `[A]` |
| 7. Stage tarball | copy, SHA-256 verify, extract | `[A/M]` (manual reconciliation) |
| 8. Render twistlock.cfg | template → disk | `[A/M]` |
| 9. Run twistlock.sh | interactive installer | `[M]` |
| 10. systemd / logrotate / backup / monitoring | overrides, logrotate, timer, node_exporter | `[A]` |
| 11. First-run wizard | admin, licence, LDAPS, retention, alerts | `[M]` |
| 6 (§6). Verification | acceptance checks | `[A/M]` |

---

## 14. Ansible role outline

Proposed repository layout under a new `prisma-twistlock-install` repo (the currently-empty `/Users/chait/vachan-homelab/projects/prisma-twistlock-install` is the natural home):

```
prisma-twistlock-install/
├── README.md                              # points at this runbook
├── ansible.cfg
├── inventory/                             # directory-per-env (dev, pre-prod-ht/vht, prod-ht/vht)
│   ├── dev/
│   │   ├── hosts.yml                      # 2 DCs × 3 VM roles
│   │   ├── group_vars/                    # env-wide + per-DC overrides
│   │   └── host_vars/                     # per-host overrides
│   └── <env>/hosts.yml                    # commented template for each non-dev env
├── group_vars/                            # cross-env defaults (apply to every env)
│   ├── all.yml
│   ├── prisma_console.yml
│   ├── prisma_scanner.yml
│   └── prisma_sandbox.yml
├── playbooks/
│   ├── 00-baseline.yml                    # Phases 2–4
│   ├── 10-podman.yml                      # Phase 5
│   ├── 20-prisma-stage.yml                # Phases 6–8
│   ├── 30-prisma-ops.yml                  # Phase 10
│   └── site.yml                           # orchestrates all of the above
└── roles/
    ├── rhel_baseline/                     # packages, SELinux, firewalld, chrony, sshd, sysctl, journald
    ├── fips_enable/                       # Phase 2 (with handled reboot)
    ├── prisma_storage/                    # Phase 4
    ├── podman_config/                     # Phase 5
    ├── prisma_tls/                        # Phase 6
    ├── prisma_stage/                      # Phase 7 (copy tarball, SHA-256, extract)
    ├── prisma_config/                     # Phase 8 (render twistlock.cfg)
    ├── prisma_systemd/                    # Phase 10.1 systemd override
    ├── prisma_logrotate/                  # Phase 10.2
    ├── prisma_backup/                     # Phase 10.3
    └── prisma_monitoring/                 # Phase 10.4 + 10.5
```

**Secret handling**: all passwords, bind credentials, licence keys, and backup tokens pulled from HashiCorp Vault via the existing `twistlock-exemption` pipeline's Vault AppRole pattern (reuse `ffddce8 Add Vault AppRole client for credential retrieval` if appropriate), never committed to this repo.

**Idempotency requirement**: every role must be runnable repeatedly on the same host with no side effects once converged. The only non-idempotent step is `twistlock.sh` itself, which the playbooks deliberately **do not** wrap — it remains manual in Phase 9.

**Tag scheme**: each role registers an Ansible tag matching its phase (`baseline`, `fips`, `storage`, `podman`, `tls`, `stage`, `config`, `systemd`, `logrotate`, `backup`, `monitoring`) so an operator can re-run a single slice of the build.

**Testing**: add a Molecule scenario with a RHEL 8.10 Vagrant / VirtualBox sandbox and a `verify.yml` that asserts each phase's acceptance criteria programmatically. Run against the lab before touching production.

### 12.7 Reference URLs
- Deploy in FIPS mode: https://docs.prismacloud.io/en/compute-edition/34/admin-guide/howto/deploy-in-fips-mode
- Console on Onebox: https://docs.prismacloud.io/en/compute-edition/34/admin-guide/install/deploy-console/console-on-onebox
- Reconfigure Twistlock: https://docs.prismacloud.io/en/compute-edition/34/admin-guide/configure/reconfigure-twistlock
- System Requirements: https://docs.prismacloud.io/en/compute-edition/34/admin-guide/install/system-requirements
- Performance Planning: https://docs.prismacloud.io/en/compute-edition/34/admin-guide/deployment-patterns/performance-planning
- Prisma Cloud Reference Architecture — Console: https://docs.paloaltonetworks.com/prisma/prisma-cloud/prisma-cloud-reference-architecture-compute/platform_components/console
- RHEL 8 FIPS enablement: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/assembly_installing-the-system-in-fips-mode_security-hardening
- RHEL 8 podman + container configuration: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/building_running_and_managing_containers/index

---

*End of runbook.*
