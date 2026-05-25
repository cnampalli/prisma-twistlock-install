# Uninstall an existing Prisma Cloud Compute Console

Remove a **manually-installed** Prisma Cloud Compute Console from a RHEL 8.10
host so the host can be re-provisioned cleanly by this automation. Run it on the
console host as root (or via `sudo`). Repeat on **each** console host (primary
and secondary).

> ⚠️ **Destructive.** This stops the Console and can delete its data. **Take a
> backup first** (§2). FIPS and the OS baseline are intentionally left intact
> (§7). Steps are written "if present" — a manual install may not have every
> component (node_exporter, backup timer, LVM data disk, etc.), so skip what
> doesn't exist.

This mirrors, in reverse, the install footprint in
[`docs/runbook.md`](runbook.md) and the roles under `roles/`. Defenders on
scanner/sandbox hosts are **separate** — see §8.

---

## 1. Capture current state (so you know what to remove)

```bash
systemctl list-units --all | grep -Ei 'twistlock|prisma|node_exporter'
podman ps -a
podman images
podman volume ls
findmnt /var/lib/twistlock 2>/dev/null            # is data on its own LVM volume?
firewall-cmd --list-ports
ls -la /var/lib/twistlock /opt/prisma-install /etc/pki/prisma /etc/prisma 2>/dev/null
```

Note the exact container name (the runbook uses `twistlock_console`) and whether
`/var/lib/twistlock` is a mounted LVM volume or a plain directory.

## 2. Back up first (strongly recommended)

If you might need the data (settings, scan history, the embedded DB), snapshot it
before deleting:

```bash
# Quietest path: stop the Console, then archive the data folder.
systemctl stop twistlock 2>/dev/null || true
tar -czf /root/twistlock-data-$(date +%Y%m%dT%H%M%SZ).tar.gz -C /var/lib twistlock
# Or, if the Console backup API/flow was set up, trigger its backup instead.
```

Copy that tarball off-box if the host itself is going away.

## 3. Stop and disable services

```bash
# Console + ops-layer units (ignore "not found" for anything not installed)
systemctl stop    twistlock prisma-backup.timer prisma-backup.service node_exporter 2>/dev/null || true
systemctl disable twistlock prisma-backup.timer prisma-backup.service node_exporter 2>/dev/null || true
```

## 4. Remove the container(s) and images

```bash
# Rootful podman (Console host). Use the name you saw in §1.
podman rm -f twistlock_console 2>/dev/null || true
podman rmi -f $(podman images -q --filter reference='*twistlock*') 2>/dev/null || true
podman volume prune -f                      # remove dangling Console volumes
# Sanity: nothing twistlock left
podman ps -a; podman images | grep -i twistlock || echo "no twistlock images"
```

## 5. Remove systemd units, drop-ins, and helper files

```bash
rm -f  /etc/systemd/system/twistlock.service
rm -rf /etc/systemd/system/twistlock.service.d          # memory/restart override (prisma_systemd)
rm -f  /etc/systemd/system/prisma-backup.service /etc/systemd/system/prisma-backup.timer
rm -f  /etc/systemd/system/node_exporter.service        # if monitoring was installed
rm -f  /usr/local/sbin/prisma-backup.sh                 # backup script (prisma_backup)
rm -rf /etc/prisma                                       # backup.env etc.
rm -f  /etc/logrotate.d/prisma-console                  # logrotate policy
rm -f  /etc/rsyslog.d/*prisma* 2>/dev/null || true      # SIEM shipping, if configured
systemctl daemon-reload
```

> Leave `disable-thp.service` (from the baseline) unless you are fully
> de-baselining the host — it is a harmless boot-time tweak, not Console state.

## 6. Remove application config + staged installer

```bash
rm -rf /opt/prisma-install            # staged + extracted tarball (prisma_stage)
rm -rf /etc/pki/prisma                # Console TLS cert/key/CA (prisma_tls)
```

### Console data — choose one

```bash
# (a) Re-installing on the SAME host and want a clean DB: wipe the data.
rm -rf /var/lib/twistlock/*           # keeps the mount; clears contents
# (b) Or remove the directory entirely (only if NOT an LVM mount — see §6.1).
# rm -rf /var/lib/twistlock
```

You already archived this in §2. The Console will recreate `/var/lib/twistlock`
on the next install.

### 6.1 Optional — tear down the dedicated data volume (LVM)

Only if `/var/lib/twistlock` is its own LVM volume (per §1) **and** you want to
reclaim the disk for a truly clean re-provision. This destroys the data disk.

```bash
umount /var/lib/twistlock
sed -i '\#/var/lib/twistlock#d' /etc/fstab           # remove the mount line
lvremove -y /dev/vg_twistlock/lv_data
vgremove -y vg_twistlock
pvremove -y /dev/sdb                                  # or: wipefs -a /dev/sdb
```

If you keep the volume, the `prisma_storage` role will detect it already exists
and reuse it on the next run.

## 7. Optional — firewall, and what to LEAVE alone

```bash
# Close the Console/Defender/exporter ports only if not re-installing soon:
firewall-cmd --permanent --remove-port=8083/tcp --remove-port=8084/tcp --remove-port=9100/tcp
firewall-cmd --reload
```

**Leave intact** (OS-level, not Console state; the install expects them):
- **FIPS mode** — do not run `fips-mode-setup --disable`. The re-install
  requires FIPS on.
- SELinux enforcing, chrony/NTP, journald caps, sysctl, podman config
  (`/etc/containers/*.conf`), and the `podman` package.

## 8. Defenders are separate

Uninstalling the Console does **not** remove Defenders deployed on scanner /
sandbox hosts (those run under docker-ce). On each such host:

```bash
# If installed via twistcli (defender install):
/path/to/twistcli defender uninstall 2>/dev/null || true
docker rm -f $(docker ps -aq --filter name=twistlock) 2>/dev/null || true
docker rmi $(docker images -q --filter reference='*twistlock*') 2>/dev/null || true
rm -f /etc/systemd/system/twistlock-defender*.service
rm -rf /var/lib/twistlock-defender
systemctl daemon-reload
```

## 9. Verify a clean slate

```bash
systemctl list-units --all | grep -Ei 'twistlock|prisma' || echo "no twistlock units"
podman ps -a; podman images | grep -i twistlock || echo "no twistlock images"
ls /etc/systemd/system/ | grep -Ei 'twistlock|prisma' || echo "no leftover units"
ls -d /opt/prisma-install /etc/pki/prisma 2>/dev/null || echo "config dirs gone"
test -e /etc/redhat-release && fips-mode-setup --check   # confirm FIPS still ON
```

Expect: no twistlock units/containers/images, config dirs gone, **FIPS still
enabled**.

## 10. Re-install with this automation

Once the host is clean (and FIPS is still on):

```bash
# CLI path (per environment):
ansible-playbook -i inventories/<env>/hosts.yml playbooks/site.yml      # Phases 2–8
# then the manual Phase 9 on the console host:
cd /opt/prisma-install/prisma_cloud_compute_edition_34_01_126
./twistlock.sh -s console | tee /root/twistlock-install.log
# then the ops layer:
ansible-playbook -i inventories/<env>/hosts.yml playbooks/30-prisma-ops.yml   # Phase 10
```

For the AAP/Controller path, see [`docs/aap-controller-setup.md`](aap-controller-setup.md).
If the data disk/volume already exists and is healthy, the `prisma_storage` role
reuses it; otherwise it rebuilds it.
