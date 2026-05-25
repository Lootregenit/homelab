# Phase 1 — Proxmox VE install (both nodes)

Install Proxmox VE 9 on both Dell Precision 3561 laptops, get them clean and
consistent, but **do not cluster yet** (that's Phase 2).

---

## BIOS prep (F2 at boot, both laptops)

| Setting | Value | Why |
|---------|-------|-----|
| Virtualization | On | Required for KVM |
| VT for Direct I/O (VT-d) | On | IOMMU / future passthrough |
| Thunderbolt → Security Level | User Authorization | Lets the TB4 net interface come up in Phase 2 |
| Thunderbolt Boot Support / Pre-boot | Off | Not booting over TB |
| Secure Boot | **On** | PVE 9 supports it; leaving it on is the more secure, current choice |
| Integrated NIC | Enabled w/ PXE | Costs nothing, enables future PXE (roadmap) |
| Boot order | USB first | For the installer |

> **Why Secure Boot ON:** the old "disable Secure Boot for Proxmox" advice is
> outdated. PVE 9 ships a signed shim + kernel. Only out-of-tree modules (e.g.
> NVIDIA) need MOK signing later. Leaving it on is defense-in-depth and a better
> story than "I turned the security features off."

---

## Installer choices

Filesystem screen → **ZFS (RAID0)**, single disk. Then **Advanced Options**:

| Option | Value | Why |
|--------|-------|-----|
| Filesystem | `zfs (RAID0)` | Snapshots + `zfs send/recv` replication later |
| ashift | 12 | 4K sectors — correct for any modern SSD/NVMe |
| compress | on | LZ4 — free performance |
| checksum | on | Bit-rot detection — never disable |
| copies | 1 | Single disk; more copies just wastes space |
| ARC max size | **4096 MiB** | Cache without starving 32 GB of VM RAM |
| hdsize | **~1800 GB** (≈100 GB below disk) | SSD over-provisioning + avoid ZFS >80%-full slowdown |

Network screen — **the important one**:

| Field | Value | Why |
|-------|-------|-----|
| Management interface | wired Ethernet (`nic0`) | Never wifi for a node |
| Hostname (FQDN) | `pve-01.infra.lootregen.com` | **Full FQDN, not short name** — certs are per-FQDN |
| IP/CIDR | `192.168.1.100/24` (pve-02: `.101`) | Static, outside DHCP pool |
| Gateway | `192.168.1.1` | Router |
| DNS | `192.168.1.1` | Router, **not** the ISP DNS the installer auto-fills |

> **Why FQDN at install time:** setting just `pve-01` (no domain) makes the TLS
> cert issue for a bare hostname, which later fails verification when another node
> connects (exactly the Phase 2 `pvecm add` failure). Set the full FQDN now.

> **Why router DNS not ISP DNS:** the installer auto-grabbed the ISP resolver from
> the laptop's DHCP lease. Hardcoding ISP DNS bypasses any future internal DNS
> (AD, Pi-hole). Point at the gateway so name resolution stays under my control.

Keymap: `pl` (Polish programmers) — a personal-tooling preference, not infra.

---

## Post-install (SSH in as root, both nodes)

PVE 9 uses the **deb822 `.sources`** format, not the old `.list` files.

```bash
# Disable enterprise + Ceph enterprise repos (deb822 format!)
sed -i 's/^Enabled: yes/Enabled: no/' /etc/apt/sources.list.d/pve-enterprise.sources
sed -i 's/^Enabled: yes/Enabled: no/' /etc/apt/sources.list.d/ceph.sources

# Add no-subscription repo
cat > /etc/apt/sources.list.d/pve-no-sub.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF

apt update && apt full-upgrade -y

# Remove the "no valid subscription" nag popup
sed -Ezi.bak "s/(Ext\.Msg\.show\(\{.*?title: gettext\('No valid sub)/void\(\{ \/\/\1/g" \
  /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy

# CRITICAL: closing the laptop lid must NOT suspend a cluster node
sed -i 's/#HandleLidSwitch=suspend/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
sed -i 's/#HandleLidSwitchDocked=suspend/HandleLidSwitchDocked=ignore/' /etc/systemd/logind.conf
sed -i 's/#HandleLidSwitchExternalPower=suspend/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf
systemctl restart systemd-logind

# IOMMU for future GPU passthrough
sed -i 's|GRUB_CMDLINE_LINUX_DEFAULT="quiet"|GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"|' /etc/default/grub
update-grub

reboot
```

> **Why `.sources` not `.list`:** Debian 12/13 migrated to the deb822 format. The
> old `sed 's/^deb/#deb/' ...pve-enterprise.list` does nothing on PVE 9 because the
> file doesn't exist — you get `401 Unauthorized` from the enterprise repo until
> you disable it via `Enabled: no` in the `.sources` file.

> **Why lid switch matters:** these are laptops. Default behaviour suspends on lid
> close — which would drop a cluster node the moment I shut the lid. `ignore` keeps
> them running headless.

Verify after reboot:
```bash
hostname -f          # pve-01.infra.lootregen.com
journalctl -p 3 -xb  # error-level logs since boot — should be empty
```

---

## 💥 Lesson — renaming a node that already has VMs

I installed the first node with the default hostname `proxmox`, created VMs, then
renamed it to `pve-01`. Proxmox's clustered config FS (`/etc/pve`) treats a
renamed node as **brand new** — it created `nodes/pve-01/` but left the VMs under
the old `nodes/proxmox/`. The VMs "disappeared" from the new node in the UI.

**Fix:**
```bash
# Move VM (and any LXC) configs from the ghost node to the real one
mv /etc/pve/nodes/proxmox/qemu-server/*.conf /etc/pve/nodes/pve-01/qemu-server/
mv /etc/pve/nodes/proxmox/lxc/*.conf /etc/pve/nodes/pve-01/lxc/ 2>/dev/null

# Delete the ghost node (its leftover files are runtime/crypto, auto-regenerated)
rm -rf /etc/pve/nodes/proxmox
```

The VM **disks** were never at risk — they live in the ZFS pool (`local-zfs`),
not in `/etc/pve`. Only the metadata pointers moved.

**Prevention:** name nodes correctly at install. If you must rename later: stop
VMs, move configs, *then* change the hostname — never the other way round.

**Renaming a node correctly (before it has VMs):**
```bash
hostnamectl set-hostname pve-01
# edit /etc/hosts: 192.168.1.100  pve-01.infra.lootregen.com pve-01
echo "pve-01.infra.lootregen.com" > /etc/mailname
reboot
```
