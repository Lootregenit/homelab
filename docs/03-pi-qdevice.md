# Phase 3 — Raspberry Pi QDevice

Add a third quorum vote so the cluster survives the loss of any single member.

> **Why:** a 2-node cluster has no tiebreaker. If either node dies, the survivor
> holds 1 of 2 votes and loses quorum — VMs won't start. A QDevice adds a 3rd
> vote without needing a 3rd full Proxmox node. A Raspberry Pi 3 is plenty.

---

## 3.1 — Pi base (already running Pi OS Lite 64-bit)

Static IP via DHCP reservation on the router (MAC → `192.168.1.60`), then:

```bash
sudo hostnamectl set-hostname qdev-01
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y corosync-qnetd openssh-server
sudo systemctl enable --now corosync-qnetd
```

Temporarily allow root SSH — `pvecm qdevice setup` needs it for the one-time
key exchange:
```bash
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo passwd root           # set a temp password
sudo systemctl restart ssh
```

---

## 3.2 — QDevice client on both nodes

```bash
# On pve-01 AND pve-02
apt install -y corosync-qdevice
```

---

## 3.3 — Register (pve-01 only)

```bash
pvecm qdevice setup 192.168.1.60
```

Prompts for the Pi's root password, exchanges keys, configures both nodes.

---

## 3.4 — Verify

```bash
pvecm status
```

Want to see:
```
Expected votes:   3
Total votes:      3
Quorum:           2
Flags:            Quorate Qdevice

    Nodeid      Votes    Qdevice Name
0x00000001          1    A,V,NMW 192.168.1.100 (local)
0x00000002          1    A,V,NMW 192.168.1.101
0x00000000          1            Qdevice
```

> **`A,V,NMW`** = Alive, Voting, Not-Master-Wins. If the two nodes can't see each
> other (network partition), whichever still reaches the QDevice keeps quorum; the
> other fences itself. Same split-brain prevention as enterprise clusters.

---

## 3.5 — Lock the Pi back down

The `corosync-qnetd` daemon talks over its own TLS on port 5403 — it does **not**
need SSH after the initial setup.

```bash
# On the Pi
sudo sed -i 's/^PermitRootLogin yes/PermitRootLogin prohibit-password/' /etc/ssh/sshd_config
sudo passwd -l root
sudo systemctl restart ssh
```

Confirm quorum still intact afterwards: `pvecm status` → still 3 votes.

---

## 3.6 — Failure simulation (the real proof)

Configuration isn't proof. Test it:

```bash
# Gracefully shut down pve-02
shutdown -h now            # on pve-02

# On pve-01:
pvecm status
#  → Total votes: 2 (pve-01 + Qdevice), Quorum: 2, Flags: Quorate
#  → VMs still start because the Pi provides the tiebreaker
```

Without the Pi this would be 1/2 = not quorate. Power pve-02 back on, watch it
rejoin (`pvecm status` → 3 votes). **This test is what turns "configured" into
"verified."**

---

## Design note

The Pi does **exactly one job** — vote on quorum. No other services on it. Single
responsibility = boring and reliable, which is exactly what a quorum device should
be. (This is also why the later NUT alert speaker lives on the laptops, not the
Pi — keep the Pi's role minimal.)
