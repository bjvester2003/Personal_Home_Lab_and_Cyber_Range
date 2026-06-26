# Home Network & Lab — Phase 1 Completion Report

## Overview

Phase 1 established the core network infrastructure for a zone-segmented homelab and cyberrange built on three Dell Optiplex 9020 nodes running Proxmox VE, a Cisco SG200-26 managed switch, and pfSense CE as the firewall/router. The goal was to replace a flat, unsegmented home network with a VLAN-segmented architecture where all traffic is gated through pfSense and each zone is isolated by default.

---

## Physical Topology (Final)

```
Internet
   │ (Coaxial)
Xfinity Gateway (10.0.0.1)
   │
Netgear G5605 (unmanaged)
   ├── Xbox Series S          ← WAN-side, not behind pfSense yet
   │
Cisco SG200-26 (10.10.10.2)
   ├── Port 1   → Netgear uplink
   ├── Port 2   → Desktop (10.10.10.100, VLAN 10 only)
   ├── Port 4   → singledecker nic0 / vmbr0 (pfSense WAN uplink) — VLAN 1 permanent
   ├── Port 5   → doubledecker nic0 / vmbr0 (1U + 10T)
   ├── Port 6   → tripledecker nic0 / vmbr0 (1U + 10T)
   ├── Port 16  → singledecker nic1 / vmbr1 (LAN trunk: 1U + 10T–70T)
   └── GE24     → spare VLAN 10 test port (10T)
```

---

## Devices

| Device | Model | Role | Management IP |
|---|---|---|---|
| singledecker.pve | Dell Optiplex 9020 | Proxmox node 1, pfSense host | 10.10.10.6 |
| doubledecker.pve | Dell Optiplex 9020 | Proxmox node 2 | 10.10.10.7 |
| tripledecker.pve | Dell Optiplex 9020 | Proxmox node 3 | 10.10.10.8 |
| Cisco SG200-26 | Cisco SG200-26 | Managed switch | 10.10.10.2 |
| pfSense VM | VM on singledecker | Firewall/router | 10.10.10.1 (MGMT) / 10.0.0.169 (WAN) |
| Desktop | Dell Precision 3630 | Admin workstation | 10.10.10.100 (Static Lease) |
| Xbox Series S | — | Gaming console | WAN-side (10.0.0.x, not yet behind pfSense) |
| Xfinity Gateway | — | ISP modem/router | 10.0.0.1 |
| Netgear G5605 | — | Unmanaged pass-through switch | N/A |

---

## Proxmox Cluster

- **Cluster name:** BurgerMart
- **Nodes:** singledecker (1), doubledecker (2), tripledecker (3)
- **Corosync:** all heartbeat traffic on VLAN 10 (10.10.10.x)
- **Quorum:** healthy, all nodes Quorate: Yes

### Per-Node Network Config (Final)

**singledecker**
- `nic0` (enp0s25) → `vmbr0` (no IP — bridges WAN to pfSense `vtnet0`) → port 4
- `nic1` (PCIe NIC) → `vmbr1` (VLAN-aware trunk, no IP) → port 16
- `vlan10` tapped off `vmbr1`: `10.10.10.6/24`, gateway `10.10.10.1`

**doubledecker**
- `nic0` → `vmbr0` (no IP) → port 5
- `vlan10` tapped off `nic0`: `10.10.10.7/24`, gateway `10.10.10.1`

**tripledecker**
- `nic0` → `vmbr0` (no IP) → port 6
- `vlan10` tapped off `nic0`: `10.10.10.8/24`, gateway `10.10.10.1`

---

## VLAN Architecture

| VLAN | Zone | Subnet | Gateway | DHCP Range | Purpose |
|---|---|---|---|---|---|
| 10 | Management | 10.10.10.0/24 | 10.10.10.1 | .101–.200 | Infrastructure admin — pfSense, Proxmox, Cisco, desktop |
| 20 | LAN | 10.10.20.0/24 | 10.10.20.1 | .101–.200 | Trusted personal devices (future) |
| 30 | DMZ | 10.10.30.0/24 | 10.10.30.1 | None (static) | Public-facing services — game servers, reverse proxy, etc. |
| 40 | Lab | 10.10.40.0/24 | 10.10.40.1 | None (static) | Isolated hacking/vuln lab — no internal access |
| 50 | IoT_Home | 10.10.50.0/24 | 10.10.50.1 | .101–.200 | Untrusted devices — Xbox (future), smart home |
| 60 | Wireless | 10.10.60.0/24 | 10.10.60.1 | .101–.200 | Laptops, phones, guests — internet only |
| 70 | Storage | 10.10.70.0/24 | 10.10.70.1 | None (static) | Future TrueNAS — MGMT and DMZ access only |

---

## pfSense Configuration

### Interfaces
- **WAN** (`vtnet0`): DHCP, `10.0.0.169`, gateway `10.0.0.1`
- **MGMT** (`vtnet1.10`): static `10.10.10.1/24`
- **LAN** (`vtnet1.20`): static `10.10.20.1/24`
- **DMZ** (`vtnet1.30`): static `10.10.30.1/24`
- **LAB** (`vtnet1.40`): static `10.10.40.1/24`
- **IOT_HOME** (`vtnet1.50`): static `10.10.50.1/24`
- **WIRELESS** (`vtnet1.60`): static `10.10.60.1/24`
- **STORAGE** (`vtnet1.70`): static `10.10.70.1/24`
- **Legacy LAN** (`vtnet1` untagged, `192.168.1.1`): **DISABLED**

### Firewall Aliases

| Alias | Type | Value(s) | Purpose |
|---|---|---|---|
| `desktop_ug4bkha` | Host | `10.10.10.101` | Admin desktop |
| `pfsense` | Host | `10.10.10.1` | pfSense gateway |
| `cisco_switch` | Host | `10.10.10.2` | Cisco SG200-26 |
| `singledecker` | Host | `10.10.10.6` | Proxmox node 1 |
| `doubledecker` | Host | `10.10.10.7` | Proxmox node 2 |
| `tripledecker` | Host | `10.10.10.8` | Proxmox node 3 |
| `burgermart` | Host | `10.10.10.6`, `.7`, `.8` | All Proxmox nodes |
| `RFC1918` | Network | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | All private ranges |

### Firewall Rules — MGMT (in order)

| # | Action | Source | Destination | Description |
|---|---|---|---|---|
| 1 | Pass | `desktop_ug4bkha` | any | Desktop full access |
| 2 | Pass | `burgermart` | MGMT subnets | Proxmox cluster heartbeat |
| 3 | Pass | `burgermart` | !RFC1918 | Proxmox internet access |
| 4 | Pass | `cisco_switch` | !RFC1918 | Cisco switch internet access |
| 5 | Block | MGMT subnets | any | Default deny |

### Firewall Rules — All other VLANs (20–70)
Default deny — no rules, no devices placed yet.

---

## Cisco SG200-26 Port Config

| Port | VLAN Config | Connected Device | Notes |
|---|---|---|---|
| 1 | 1U | Netgear uplink | Permanent — never change |
| 2 | 10U | Desktop | MGMT only |
| 4 | 1U | singledecker nic0 (pfSense WAN) | Permanent — never change |
| 5 | 1U + 10T | doubledecker nic0 | 1U kept as PVID per SG200 requirement |
| 6 | 1U + 10T | tripledecker nic0 | 1U kept as PVID per SG200 requirement |
| 16 | 1U + 10T–70T | singledecker nic1 (pfSense LAN trunk) | 1U inert (legacy LAN disabled in pfSense) |
| GE24 | 10T | Spare test port | Left configured, available as fallback |

---

## Issues Encountered & Resolutions

### 1. pfSense installer — CD drive boot hang
**Symptom:** VM stuck on `cd0: attempt to query device size failed: NOT READY, medium not present` after install.
**Cause:** I missed the "press 'y' to continue" line.
**Fix:** I pressed 'y'.

### 2. pfSense boot hang on `rc.linkup`
**Symptom:** pfSense hung at `ignoring link event during boot sequence` indefinitely with two NICs attached.
**Cause:** Same as above... seriously i spent an hour on this because why would i bother to read the logs a page up and not just the logs in front of my face.
**Fix:** I stopped being an idiot.

### 3. pfSense web GUI inaccessible from desktop
**Symptom:** Could not reach pfSense at `10.0.0.169` from desktop on same subnet.
**Cause:** pfSense blocks all inbound connections on WAN by default, including GUI access. Desktop was on WAN-side subnet.
**Fix:** Set desktop to static `192.168.1.50/24` to reach pfSense via its default LAN interface at `192.168.1.1`.

### 4. Cisco switch lost after factory reset
**Symptom:** Switch unreachable at whatever ip it went to after reset — desktop on `10.0.0.x`.
**Cause:** Subnet mismatch — switch fell back to some stupid default config and got a `192.168.1.x` address.
**Fix:** Found the switch at `192.168.1.250` after searching for like 30 minutes.

### 5. Desktop lockout during first VLAN 10 migration attempt
**Symptom:** Lost all access to pfSense GUI, Proxmox, and internet after moving port 2 to VLAN 10.
**Cause:** Two simultaneous issues — firewall rule used "MGMT address" (pfSense's own IP only) instead of "MGMT net" (full subnet), AND port 2 was moved to VLAN 10 without a fallback access path.
**Fix:** Used laptop on a separate port to revert the Cisco switch config. Adopted a safer procedure: test on a spare port first, leave main desktop port untouched until VLAN 10 is proven working.

### 6. singledecker vlan10 tapped off wrong parent (nic1 vs vmbr1)
**Symptom:** Adding `vlan10` interface with parent `nic1` caused pfSense to lose `10.10.10.1` — desktop lost internet and gateway access. Only singledecker itself (`10.10.10.6`) remained reachable.
**Cause:** `nic1` is enslaved into `vmbr1`. Creating a VLAN device directly on `nic1` made it compete with `vmbr1` for tagged VLAN 10 frames — the host grabbed them before they reached pfSense's VM.
**Fix:** Deleted the `vlan10` device, recreated it with parent `vmbr1` instead of `nic1`. VLAN devices on bridged NICs must always tap off the bridge, not the raw NIC.

### 7. Proxmox cluster quorum broken after singledecker IP removal
**Symptom:** Datacenter view showed doubledecker and tripledecker as "unavailable" immediately after removing `10.0.0.151` from singledecker.
**Cause:** Corosync's `ring0_addr` for singledecker was still `10.0.0.151` — removing the IP broke the cluster heartbeat before corosync was updated to use the new address.
**Fix:** Manually updated `/etc/pve/corosync.conf` and `/etc/corosync/corosync.conf` on all nodes with new `ring0_addr: 10.10.10.6`, incremented `config_version`, restarted corosync simultaneously across all nodes. Lesson: always migrate corosync link address first, verify quorum, then remove old IP.

### 8. Desktop DHCP static mapping not assigning correct IP
**Symptom:** Desktop consistently received `10.10.10.101` instead of the reserved `10.10.10.100` despite correct MAC/IP mapping in pfSense.
**Cause:** Active lease cached on both client and server sides — pfSense honored the existing lease over the static mapping.
**Resolution:** Accepted `.101` as desktop's address going forward. Static mapping issue was not worth further troubleshooting time. All firewall rules reference the desktop via alias, so the specific IP is easily updated if resolved later.
**NOTE:** This thing fixed itself. I went downstairs for like 30 minutes and came back to no internet. Checked some things and i had gotten the `.100` address. No access to pfsense though so the laptop and rescue port came back out. Easy fix, swap the firewall alias from `.101` to `.100`. Bob's your uncle

### 9. MGMT firewall rules not loading after disabling legacy LAN interface
**Symptom:** After disabling `192.168.1.1` (legacy LAN) in pfSense, desktop lost access to `10.10.10.1` and internet despite MGMT interface showing UP in `ifconfig`.
**Cause:** pfSense's packet filter (`pf`) left in an inconsistent state after a live interface disable — `pfctl -sr` showed no MGMT rules loaded in the active ruleset.
**Fix:** Rebooted pfSense VM. Clean boot reloaded all rules from saved config correctly.

### 10. Port 16 VLAN 1 removal breaking connectivity
**Symptom:** Removing VLAN 1 from port 16 caused desktop to lose pfSense access and internet.
**Cause:** SG200-26 requires every port to have a PVID (untagged VLAN). Removing VLAN 1 without cleanly reassigning PVID to VLAN 10 left the port in an inconsistent state. Conflict between VLAN 10 being both PVID and tagged simultaneously may also have contributed.
**Fix:** Restored `1U` on port 16. Since the legacy LAN interface is disabled in pfSense, untagged VLAN 1 on port 16 is inert — no interface consumes it. Left as permanent config to satisfy SG200 PVID requirement.

---

## Known Gaps / Deferred Items

| Item | Priority | Notes |
|---|---|---|
| Xbox behind pfSense on VLAN 50 | Medium | Currently on WAN-side Netgear, unmanaged by pfSense. Requires moving Xbox cable to Cisco switch, configuring port as VLAN 50 access port. |
| Domain name purchase | Medium | Phase 0 blocker not yet resolved. Required for Cloudflare DDNS and public-facing services in Phase 2+. |
| vmbr0 VLAN-awareness on doubledecker/tripledecker | Medium | Currently not VLAN-aware. Required before placing VMs on VLANs other than 10 on those nodes. Switch ports 5/6 will also need additional VLAN tags. |
| pfSense legacy LAN port 16 cleanup | Low | `1U` on port 16 is inert but architecturally untidy. Future cleanup once a cleaner solution for SG200 PVID requirement is found. |
| Desktop dual-NIC / home network access | Low | Desktop currently MGMT-only. Plan is to add a second NIC (USB or PCIe) for simultaneous home network access rather than enabling Hyper-V (conflicts with VirtualBox). |
| MGMT firewall rules — tighten pfSense's own outbound | Low | pfSense-originated traffic bypasses interface rules entirely. Consider adding floating rules to restrict pfSense's own outbound if needed. |
| pfSense on dedicated hardware | Long-term | pfSense as a VM on singledecker is an architectural risk — if singledecker goes down, all routing and firewall dies with it. Worth planning a dedicated physical box eventually. |
| Desktop DHCP static mapping | Low | Desktop receives `.101` instead of reserved `.100`. Cosmetic issue — all rules use aliases. |

---

## Phase 2 Preview

Phase 2 focuses on monitoring and observability:
- Log server (syslog aggregation from pfSense, Proxmox, Cisco)
- SIEM deployment
- Suricata IDS on pfSense
- Network traffic visibility across all VLANs

All VLANs 20–70 are currently empty with default-deny firewall rules — they are ready to receive services as Phase 2 and beyond progresses.

## Disclaimer

This document was generated by and instance of Claude Sonnet 4.6. It was reviewed and modified as necessary. It's authenticity and accuracy to the content therein has been verified and approved.

-BV