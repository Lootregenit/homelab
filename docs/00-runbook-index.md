# Homelab Runbook — `lab-zrh-pve-01`

Detailed, rebuild-from-scratch documentation for my two-node Proxmox cluster.
Each phase file explains **what I did and why**, with secrets redacted.

> Secrets (passwords, PBS fingerprints) are placeholders like `<MASTER_PASS>`.
> Real values live in Vaultwarden under *"homelab"* entries. Never commit them.

---

## Index

| # | Phase | File |
|---|-------|------|
| 00 | Runbook index, conventions, host table | this file |
| 01 | Proxmox VE install (both nodes) | [`01-proxmox-install.md`](01-proxmox-install.md) |
| 02 | Cluster + Thunderbolt ring | [`02-cluster-thunderbolt.md`](02-cluster-thunderbolt.md) |
| 03 | Raspberry Pi QDevice | [`03-pi-qdevice.md`](03-pi-qdevice.md) |
| 04 | PBS + NAS backups | [`04-pbs-nas-backups.md`](04-pbs-nas-backups.md) |
| 04.5 | NUT power resilience | [`04.5-nut-power.md`](04.5-nut-power.md) |
| 99 | Naming standard + IP plan | [`99-naming-and-ip-plan.md`](99-naming-and-ip-plan.md) |

---

## Host inventory

| Host | Role | IP | Hardware |
|------|------|-----|----------|
| `pve-01.infra.lootregen.com` | Proxmox node (primary) | 192.168.1.100 | Dell Precision 3561 |
| `pve-02.infra.lootregen.com` | Proxmox node (secondary) | 192.168.1.101 | Dell Precision 3561 |
| `qdev-01.infra.lootregen.com` | Corosync QDevice | 192.168.1.60 | Raspberry Pi 3 |
| `pbs-01.infra.lootregen.com` | Proxmox Backup Server (VM 110) | 192.168.1.110 | VM on cluster |
| `nas-01.infra.lootregen.com` | OpenMediaVault NAS | 192.168.1.50 | Intel N100, 8 GB, 2× 4 TB + 256 GB NVMe |

**Cluster name:** `lab-zrh-pve-01`
**TB4 ring subnet:** 10.10.10.0/30 (pve-01 = .1, pve-02 = .2)

---

## Rebuild order

If rebuilding from zero, follow the phases in order — each depends on the last:

1. **Phase 1** — install both nodes (don't cluster yet)
2. **Phase 2** — Thunderbolt ring first, *then* form the cluster
3. **Phase 3** — add the Pi QDevice (cluster is fragile until this exists)
4. **Phase 4** — NAS export + PBS VM + backup/restore test
5. **Phase 4.5** — NUT on nodes, then clients
6. Roadmap items (VLANs, PXE, IaC, monitoring) — see main README

---

## Conventions (the rules I held to)

- **Naming:** `<role>-<NN>.<zone>.lootregen.com`; cluster `<env>-<loc>-<purpose>-<NN>`
- **VMID = IP last octet** where practical (VM 110 → .110)
- **One source of truth:** IP reservations on the router; passwords defined once
- **FQDN everywhere** in cluster commands — never raw IPs (TLS certs are per-FQDN)
- **Verify, don't assume:** every critical step has an explicit proof
- **Least privilege:** service accounts get the minimum role needed
- **Redact secrets** in docs; real values in Vaultwarden

See [`99-naming-and-ip-plan.md`](99-naming-and-ip-plan.md) for the full standard.

---

## Pre-flight checklist for any new host/VM

- [ ] Role abbreviation chosen (`pve`, `pbs`, `dc`, `k8s`, `docker`, `nas`, `qdev`)
- [ ] VMID from correct range (110–149 infra, 150–199 workload)
- [ ] IP matching VMID convention
- [ ] DNS zone chosen (`infra`, `ad`, `lab`)
- [ ] FQDN set in installer (not the default), **not** just the short name
- [ ] DNS server set to router (`192.168.1.1`), not ISP
- [ ] Router DHCP reservation created for the MAC
- [ ] `/etc/hosts` updated on cluster nodes
- [ ] Added to host inventory above
