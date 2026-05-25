# Naming Standard + IP Plan

The conventions every host, VM, and address follows. Adopted up front so the
homelab stays auditable as it grows.

---

## DNS zones (internal-only)

| Zone | Purpose |
|------|---------|
| `infra.lootregen.com` | Physical hosts, hypervisors, storage, management |
| `ad.lootregen.com` | Active Directory domain (future) |
| `lab.lootregen.com` | Application services (Grafana, etc., future) |

> These are **internal-only** subdomains of a real owned domain — never published
> to public DNS. This is the production pattern: one real domain, internal
> subzones resolved only by internal DNS. Real domain (not `.local`/`.lan`)
> because AD and Let's Encrypt DNS-01 both require a real, ownable domain.

---

## Host naming

```
<role>-<NN>.<zone>.lootregen.com
```

Roles: `pve` (Proxmox node), `pbs` (backup server), `qdev` (quorum device),
`nas` (storage), `dc` (domain controller), `k8s`, `docker`.

Zero-padded numbers (`01`, not `1`) so they sort correctly.

---

## Cluster naming

```
<env>-<location>-<purpose>-<NN>
```

- env: `prd`, `stg`, `dev`, `tst`, `lab`, `dr`
- location: 3-letter IATA code (`zrh` = Zurich)
- purpose: `pve`, `k8s`, `vsphere`
- Constraints: **max 15 chars**, lowercase, hyphens only, hard to change later

This cluster: **`lab-zrh-pve-01`** (15 chars exactly).

---

## VM numbering

| Range | Purpose |
|-------|---------|
| 100–109 | Legacy VMs (pre-naming-plan, not renumbered to avoid disruption) |
| 110–149 | Infrastructure VMs (PBS, AD, monitoring) |
| 150–199 | Workload VMs (Docker hosts, K8s nodes, apps) |
| 200–299 | Templates / snapshots |
| 300+ | Scratch / experimental |

**Convention: VMID last digits = IP last octet** where practical.
e.g. VMID 110 → 192.168.1.110 → `pbs-01`. Any VMID in the UI implies its address
and role without checking a wiki.

---

## IP plan — 192.168.1.0/24

```
.1            router / gateway
.2–.49        DHCP pool (laptops, phones, IoT)
.50–.59       NAS / storage infrastructure
   .50        nas-01 (OMV)
.60–.69       cluster support
   .60        qdev-01 (Pi QDevice)
.100–.109     Proxmox nodes
   .100       pve-01
   .101       pve-02
.110–.149     infrastructure VMs
   .110       pbs-01 (Proxmox Backup Server)
.150–.199     workload VMs (Docker, K8s, AD, ...)
```

TB4 cluster ring (separate, point-to-point): **10.10.10.0/30** — pve-01 `.1`,
pve-02 `.2`.

> **Why subnetted by purpose:** this is small-scale IPAM. Ranges grouped by role
> mean an IP tells you what a device is. Same discipline as enterprise IP
> management, and it makes the future VLAN migration (roadmap) a clean mapping
> (each range → a VLAN).

---

## Personal vs infrastructure config

A distinction worth keeping: **standardize infrastructure, personalize the
interactive layer.**

- **Infrastructure** (team's, follows the cluster): hostnames, FQDNs, IP plan,
  network config — strict standard.
- **Personal** (mine, follows me): keymap (`pl` Polish programmers), shell,
  editor — my preference, not subject to the standard.

---

## Secrets handling

- Real passwords / fingerprints live in **Vaultwarden**, never in docs or git.
- Docs use placeholders: `<MASTER_PASS>`, `<CLIENT_PASS>`, `<PBS_FINGERPRINT>`.
- Config templates committed as `*.example` with placeholders.
- `git-secrets` recommended pre-commit to block accidental secret commits.
