# Phase 2 — Cluster + Thunderbolt ring

Link the two laptops directly over Thunderbolt 4 and form the cluster with
**two Corosync rings** (LAN + TB4).

> **Order matters:** get Thunderbolt networking up and tested *first*, then
> create the cluster with both rings in one shot. Adding ring1 after clustering is
> possible but messier.

---

## 2.1 — `/etc/hosts` on both nodes

So the nodes resolve each other by name before any DNS exists:

```
127.0.0.1       localhost.localdomain localhost
192.168.1.100   pve-01.infra.lootregen.com pve-01
192.168.1.101   pve-02.infra.lootregen.com pve-02

::1     localhost.ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Test: `ping -c2 pve-02` from pve-01 and vice-versa.

---

## 2.2 — Thunderbolt networking (both nodes)

Plug the TB4 cable between any USB-C port on each laptop (both ports are full
TB4 on the 3561).

```bash
# Load the TB networking module now + persist it
modprobe thunderbolt-net
echo "thunderbolt-net" > /etc/modules-load.d/thunderbolt.conf

# Confirm the interface appears (may show DOWN — that's fine here)
ip link | grep -i thunder
```

> If `thunderbolt0` doesn't appear, the BIOS Thunderbolt security blocked it —
> set Security Level to *User Authorization* and accept the device on first plug-in.

Configure the point-to-point `/30` link. **pve-01** `/etc/network/interfaces`:
```
auto thunderbolt0
iface thunderbolt0 inet static
    address 10.10.10.1/30
    mtu 65520
```
**pve-02:**
```
auto thunderbolt0
iface thunderbolt0 inet static
    address 10.10.10.2/30
    mtu 65520
```

Apply + test:
```bash
ifreload -a
ping -c3 10.10.10.2          # from pve-01
# Prove it's real TB4, not USB fallback (giant MTU must pass):
ping -M do -s 65000 -c3 10.10.10.2
```

> **Why MTU 65520:** Thunderbolt networking supports jumbo frames up to 65520.
> If the giant-packet ping succeeds, the link negotiated as true TB (USB fallback
> caps at 1500). This is also how I later confirmed the cheap cable was genuinely
> running TB4.

---

## 2.3 — Create the cluster (pve-01 ONLY)

```bash
pvecm create lab-zrh-pve-01 \
  --link0 address=192.168.1.100,priority=1 \
  --link1 address=10.10.10.1,priority=0
```

- `--link0` = LAN ring, priority **1**
- `--link1` = TB4 ring, priority **0**

> **Corosync priority is inverted:** *lower number = higher priority*. So
> priority 0 (TB4) is primary, priority 1 (LAN) is failover. Two independent rings
> means a single network path failing doesn't break cluster comms.

> **Cluster naming:** `lab-zrh-pve-01` = env(lab) - location(zrh) - purpose(pve) -
> NN(01). Max 15 chars, lowercase, hyphens only. **Cannot be changed easily after
> creation** — chosen deliberately up front.

---

## 2.4 — Join pve-02

```bash
# On pve-02 — join BY HOSTNAME, not IP
pvecm add pve-01 \
  --link0 address=192.168.1.101,priority=1 \
  --link1 address=10.10.10.2,priority=0
```

Prompts for pve-01's root password + SSH fingerprint (`yes`).

> ### 💥 Lesson — `pvecm add` by IP fails TLS
> First attempt used `pvecm add 192.168.1.100` → `500 Can't connect ... (hostname
> verification failed)`. Proxmox's API cert is issued for the FQDN; connecting by
> IP fails the TLS hostname check. **Fix:** `/etc/hosts` resolves the name (2.1),
> then join by **hostname**. Rule: always FQDN, never raw IP in cluster commands.

---

## 2.5 — Verify

```bash
pvecm status              # Nodes: 2, Quorate: Yes
corosync-cfgtool -s       # LINK 0 and LINK 1 both "connected" for both nodes
```

Both rings showing `connected` = TB4 is an active Corosync link.

---

## 2.6 — Migration network

Datacenter → Options → Migration Settings:
- **Network:** `10.10.10.0/30` (the **network** address, not a host IP)
- **Type:** `secure` (left as default)

> **Why network address not host:** entering `10.10.10.2/30` works from one node's
> perspective but breaks from the other. The network ID `10.10.10.0/30` resolves
> correctly from both sides.

> **Why `secure` not `insecure`:** `insecure` is faster (no SSH wrap, parallel
> streams) but has known port-race issues and the PVE 9 UI locked the dropdown to
> `secure` anyway. For a 2-node lab with infrequent migrations, the SSH overhead
> is irrelevant — durability/simplicity over a speed gain I'd never notice.

---

## 2.7 — Speed test (optional but worth it)

```bash
apt install -y iperf3        # answer "No" to run-as-daemon
# pve-02: iperf3 -s
# pve-01: iperf3 -c 10.10.10.2 -t 30 -P 4   (use -P 4 for real throughput)
```

Measured ~15 Gbit/s single-stream (CPU-bound on one core). MTU 65520 + `bolt`
confirmed true TB4. Decided **not** to chase the theoretical 40 — the link is
nowhere near a bottleneck for cluster heartbeat or occasional migration.

---

## ⚠️ Quorum warning

After this phase the cluster works **only while both nodes are up**. Lose one and
the survivor has 1/2 votes → not quorate → won't start VMs. This is fixed in
Phase 3 (Pi QDevice). Until then, don't casually reboot a node expecting the
other to carry on.
