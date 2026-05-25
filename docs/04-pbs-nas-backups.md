# Phase 4 — Backups: Proxmox Backup Server + NAS

Stand up Proxmox Backup Server as a VM, back it onto the existing OpenMediaVault
NAS over NFS, and — critically — **prove a restore works**.

> The NAS pre-existed this project, so this phase is about *integrating an
> inherited system*, not building greenfield — which is most real platform work.

---

## The NAS as inherited

`nas-01` — Intel N100, OpenMediaVault 7. Storage stack I chose to **keep, not
rebuild**:

```
2× 4 TB HDD  → mdadm RAID1 (md0, mirror)
256 GB NVMe  → bcache caching layer (reads/writes accelerated)
             → /dev/bcache0 → ext4 → ~3.6 TB usable
```

> **Why keep it:** it works, holds ~1.1 TB of existing data, and bcache+RAID1 is a
> block-level analogue of ZFS L2ARC + mirror. Wiping a working store to chase
> architectural purity is how people lose data and weekends. Documented the *why*
> instead.

---

## 4.1 — Make the NAS reliable for storage

- **Static IP via router DHCP reservation** (MAC → `192.168.1.50`). One source of
  truth; survives an OMV reinstall. Leave OMV's interface on DHCP — don't double
  up with an OMV-side static or they fight.
- **Hostname** `nas-01`, domain `infra.lootregen.com` (Network → General, *not*
  the Workbench page which only configures the web UI itself).
- Add to `/etc/hosts` on both PVE nodes:
  `192.168.1.50  nas-01.infra.lootregen.com nas-01`

---

## 4.2 — NFS export for PBS (not SMB)

In OMV: create a **Shared Folder** `pbs-store` on the existing filesystem, then
**Services → NFS** → enable, and add a share:

| Field | Value |
|-------|-------|
| Shared folder | `pbs-store` |
| Client | `192.168.1.100,192.168.1.101,192.168.1.110` (cluster + PBS only) |
| Privilege | read/write |
| Extra options | `subtree_check,insecure,no_root_squash` |

> **Why NFS, not SMB:** PBS's datastore uses chunked content-addressable storage,
> hard links for dedup, extended attributes, and atomic renames — **none work
> reliably over SMB**. SMB also breaks sparse files and fsync for VM disks. NFS is
> the only correct choice for hypervisor/backup storage. SMB stays for *client*
> file shares only.

> **Why IP-restricted client list, not `192.168.1.0/24`:** only the cluster and
> PBS need this export. Restricting to specific IPs means a compromised IoT/guest
> device can't even mount the backup store. Biggest-value, lowest-effort hardening.

> **Why `no_root_squash`:** PBS runs as root and must own its chunk files. Squash
> would make every write owned by `nobody` and break it. Safe here *because* the
> export is IP-restricted to trusted hosts.

> **Why NOT `async`:** `async` ACKs writes before they hit disk — faster, but a
> NAS power loss loses in-flight writes, and there's no UPS yet. For backup
> integrity, durability wins. Revisit if a UPS is added. (PBS is more forgiving
> than VM disks here — chunks are checksummed and old backups survive — but `sync`
> is the correct default.)

> **Why NOT `sec=sys` listed explicitly:** it's the default; Kerberos NFS
> (`krb5`) would need a KDC I don't have yet. Listing the default adds noise.

Test the mount from a node before trusting it:
```bash
mount -t nfs nas-01:/export/pbs-store /mnt/test
df -h /mnt/test                       # ~3.6 TB
touch /mnt/test/x && ls -l /mnt/test/x  # owned by root:root → no_root_squash OK
umount /mnt/test
```

---

## 4.3 — PBS VM (`pbs-01`, VMID 110)

Create VM per the naming/IP convention (VMID 110 → `.110`):

| Setting | Value | Why |
|---------|-------|-----|
| Name / VMID | `pbs-01` / 110 | Infra range 110–149; VMID = IP octet |
| Machine / BIOS | q35 / OVMF (UEFI) + EFI disk | Modern default |
| Disk | 32 GB on `local-zfs`, VirtIO SCSI, discard + SSD emul | OS only; data lives on NAS |
| CPU / RAM | 2 cores host / 4096 MB (min 2048, balloon) | Light service |
| NIC | VirtIO on vmbr0 | — |

PBS installer: ext4, hostname `pbs-01.infra.lootregen.com` (full FQDN!), IP
`192.168.1.110/24`, gateway `.1`, **DNS `192.168.1.1`** (not ISP).

> **Why ext4 inside PBS, not ZFS:** PBS does its own chunk checksumming and dedup.
> ZFS on a single virtual disk would just add overhead.

---

## 4.4 — Mount NAS inside PBS

```bash
apt install -y nfs-common
mkdir -p /mnt/nas-pbs
echo "192.168.1.50 nas-01.infra.lootregen.com nas-01" >> /etc/hosts

# Test, then persist
mount -t nfs nas-01:/export/pbs-store /mnt/nas-pbs && df -h /mnt/nas-pbs
cat >> /etc/fstab <<'EOF'
nas-01:/export/pbs-store  /mnt/nas-pbs  nfs  defaults,_netdev,nofail,soft,timeo=30  0  0
EOF
mount -a
```

> `_netdev` waits for network, `nofail` won't block boot if NAS is down, `soft` +
> `timeo=30` fails NFS ops instead of hanging forever.

Reboot PBS, confirm `df -h /mnt/nas-pbs` still shows the NAS — fstab persistence
matters or backups break after the next restart.

---

## 4.5 — Datastore + least-privilege user

PBS web UI (`https://192.168.1.110:8007`, port **8007**):

- Repos: disable enterprise, add `pbs-no-subscription`, update, remove nag popup.
- **Datastore** → Add: name `nas-store`, path `/mnt/nas-pbs`, GC schedule daily 03:30.
- **Access Control** → add user `pveuser@pbs`, grant **`DatastoreBackup`** on
  `/datastore/nas-store`.

> **Why a dedicated least-privilege user:** the credential Proxmox stores lives on
> the cluster nodes — the most exposed machines. `DatastoreBackup` can write +
> restore but **cannot delete**. If a node is compromised, the attacker can't wipe
> the backups (append-only / immutable pattern). Space is reclaimed by scheduled
> prune + GC run by the admin, never by giving delete rights to the service
> account. This is what enterprises pay for (S3 Object Lock, Veeam immutability).

Get the fingerprint for the next step:
```bash
proxmox-backup-manager cert info | grep -i fingerprint
```

---

## 4.6 — Connect Proxmox to PBS

Proxmox → Datacenter → Storage → Add → Proxmox Backup Server:

| Field | Value |
|-------|-------|
| ID | `pbs` |
| Server | `192.168.1.110` |
| Username | `pveuser@pbs` |
| Password | `<PBS_PVEUSER_PASS>` |
| Datastore | `nas-store` |
| **Namespace** | `lab-zrh-pve-01` |
| Fingerprint | `<PBS_FINGERPRINT>` |
| Backup Retention tab | **leave empty** (PBS owns prune) |
| Encryption tab | None (for now) |

> **Why set the Namespace:** PBS auto-files cluster backups under a namespace
> matching the cluster name. If the storage is left on `Root`, the Backup tab
> looks in the wrong place and shows empty even though backups exist. Match it to
> `lab-zrh-pve-01`.

> **Why retention empty here:** prune is configured on the PBS side. Setting it in
> both places makes them fight — one source of truth.

> **Why no encryption yet:** client-side encryption means *lose the key = lose
> every backup, permanently*. Defer until key management is solid (key backed up
> in Vaultwarden). On a private LAN the risk is low.

---

## 4.7 — Schedules

- **Backup job** (Proxmox → Datacenter → Backup): all VMs, daily 02:00, snapshot
  mode, ZSTD, storage `pbs`.
- **Prune** (PBS → datastore → Prune & GC): keep-last 3, daily 7, weekly 4,
  monthly 3.
- **GC**: daily 03:30 (set at datastore creation).
- **Verify** (PBS → Verify Jobs): weekly, re-verify chunks > 7 days.

> **Why verify:** re-reads every chunk and re-checks its SHA-256 to catch silent
> bit-rot *before* I ever need a restore. A backup that's quietly corrupted looks
> fine in the UI until the day it matters.

> **No sync job yet:** sync replicates to a *second* PBS for offsite 3-2-1. I only
> have one. Parked on the roadmap (friend's PBS over VPN, or Backblaze B2).

---

## 💥 Lesson — a "successful" backup that backed up nothing

First backup finished in **1 second**, reported success. The log:
```
exclude disk 'scsi0' ... (backup=no)
backup contains no disks
starting diskless backup
```
The VM disk had a `backup=0` flag, so PBS backed up only the config. The job
*succeeded* — it just wasn't *useful*.

**Fix:**
```bash
qm config 100 | grep -E 'scsi|virtio|sata'   # find the disk + the backup=0
qm set 100 --scsi0 local-zfs:vm-100-disk-0,backup=1,size=64G
```
Re-run → log now shows `backup contains 64 GiB` and a realistic duration.

**Takeaways:**
1. **Job success ≠ useful backup.** Check duration and the `contains X GiB` line.
2. Always check disk `backup=` flags on inherited/old VMs.
3. This is *why the restore test is mandatory* — see below.

> Bonus gotcha: the backup appeared in `pvesm list pbs --vmid 100` (CLI) but not
> in the VM Backup *tab* — a stale-UI / namespace-view issue, not data loss.
> `pvesm list` is the source of truth; hard-refresh the UI / set the storage
> namespace.

---

## 4.8 — Restore test (the only test that matters)

```bash
# Restore the REAL (64 GiB) backup to a throwaway VMID
# Proxmox UI: VM 100 → Backup → select backup → Restore → target VMID 999
qm start 999          # boot it, confirm it's actually the VM
qm destroy 999 --purge  # clean up once verified
```

A backup you've never restored is a hope, not a backup. Only after VM 999 boots
from the restored data is Phase 4 genuinely complete.
