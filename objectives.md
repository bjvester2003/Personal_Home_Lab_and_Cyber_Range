# Home Network & Lab — Master Design Document

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Current State](#2-current-state)
3. [Goals & End State](#3-goals--end-state)
4. [Core Design Principles](#4-core-design-principles)
5. [Network Architecture](#5-network-architecture)
6. [VLAN Design](#6-vlan-design)
7. [Physical Topology](#7-physical-topology)
8. [Proxmox Cluster & VM Layout](#8-proxmox-cluster--vm-layout)
9. [Key Technology Decisions](#9-key-technology-decisions)
10. [Major Risks & Considerations](#10-major-risks--considerations)
11. [Build Order](#11-build-order)
12. [Documentation Standards](#12-documentation-standards)

---

## 1. Project Overview

This document describes the design, architecture, and build plan for a personal home network and security lab. The project serves two concurrent purposes:

- **Home Infrastructure** — reliable, secure hosting of personal services including game servers, a personal website, mail, media, and cloud storage
- **Security Lab** — a dedicated environment for hands-on practice in network security, penetration testing, and enterprise IT simulation

The project is intentionally designed as a **resume-quality portfolio piece**, with an emphasis on proper system design, security principles, documentation, and real-world applicability.

---

## 2. Current State

### Hardware

| Device | Role | Location | Connection |
|---|---|---|---|
| Xfinity Gateway | ISP modem/router | East side, family room | WAN |
| Netgear Switch (unmanaged) | Bedroom entry switch | Under desk, SW bedroom | Cat6 from gateway (~40ft) |
| Xbox | Gaming console | Atop desk | Netgear switch |
| Cisco Switch (managed) | Core managed switch | NE corner, bedroom | Cat6 from Netgear (~30ft) |
| Dell Optiplex 9020 (x3) | Proxmox cluster nodes | NE corner, bedroom | Cisco switch |
| Personal Desktop | Primary workstation | Under desk, SW bedroom | Cisco switch |
| Personal Laptop | Mobile workstation | Wireless | Xfinity gateway (WiFi) |

### Proxmox Cluster Specs (per node)
- **RAM:** 32GB DDR3 1600MHz
- **CPU:** 8 cores
- **Storage:** 2x 500GB drives, RAID 1 (500GB usable per node)
- **Total cluster resources:** 96GB RAM, 24 CPU cores, ~1.5TB usable storage

### Current Network State
- Flat network — all devices share one broadcast domain
- No VLAN segmentation
- No dedicated firewall
- Xfinity gateway handling all NAT, DHCP, and DNS
- No monitoring or logging infrastructure

---

## 3. Goals & End State

### Public-Facing Services
- Minecraft server
- Arma 3 server
- Personal website
- Mail server
- Nextcloud (personal cloud)
- Jellyfin (media server — possibly VPN-gated rather than fully public)

### Internal / Lab Services
- Hacking lab (Kali Linux)
- Active Directory lab (Game of Active Directory)
- Monitoring and SIEM stack
- Proxmox Backup Server
- K3s Kubernetes cluster (learning)

### Future Hardware
- Dedicated NAS (TrueNAS or Arch Linux) for file and media storage

---

## 4. Core Design Principles

### Zone-Based Security Model
Every device and service lives in a defined zone of trust. Traffic between zones is explicitly permitted or denied by firewall rules — never implicitly trusted.

### Least Privilege
No zone or device gets more access than it strictly needs. A compromised DMZ server must never be able to reach personal devices. A hacking lab VM must never be able to reach production infrastructure.

### Defense in Depth
Security is enforced at multiple layers: physical, network (VLAN + firewall), host (hardened OS, minimal services), and application (TLS, 2FA, strong auth).

### Visibility Before Exposure
Logging and monitoring infrastructure is built before any service is exposed to the internet. You cannot defend what you cannot see.

### One Change at a Time
Changes are made incrementally and validated before proceeding. Snapshots are taken before every major change in Proxmox. This makes troubleshooting tractable.

---

## 5. Network Architecture

### Zone Map

| Zone | VLAN | Contents | Trust Level |
|---|---|---|---|
| WAN | — | Xfinity / Internet | Untrusted |
| Management | 10 | Desktop (host OS), Proxmox interfaces, Cisco mgmt, pfSense admin | Highest |
| LAN | 20 | Reserved for future trusted devices | High |
| DMZ | 30 | Game servers, website, mail, Nextcloud, reverse proxy | Semi-trusted |
| Lab | 40 | Kali VMs, AD lab, vulnerable VMs | Untrusted internally |
| IoT / Home | 50 | Xbox, smart home devices | Internet only |
| Wireless | 60 | Laptop, phones | Internet + VPN access only |
| Storage | 70 | NAS / file server (future) | Restricted |

### Inter-Zone Firewall Policy (Summary)

| Source | Destination | Policy |
|---|---|---|
| WAN | DMZ | Allowed on specific ports only (80, 443, 25565, etc.) |
| WAN | Any other zone | Deny all |
| DMZ | LAN / Management | **Deny all** — critical rule |
| DMZ | Internet | Allowed with scrutiny |
| LAN / Management | DMZ | Allowed (for management) |
| Lab | Any other zone | Deny all |
| Lab | Internet | Blocked or strictly whitelisted |
| Wireless | LAN / Management | Deny — VPN required |
| Wireless | DMZ | VPN on-ramp only |
| Management | All zones | Allowed (admin access) |

### Desktop Special Role
The desktop host OS lives on the Management VLAN and has explicit firewall rules permitting administrative access to all zones. Kali and other attack VMs run on the desktop or Proxmox and are **bridged to the Lab VLAN** when in use — the host OS is never itself "in" the lab.

### Laptop / Wireless Policy
The laptop is treated as an untrusted device. Access to internal services requires VPN (WireGuard) into a permitted zone. This applies even though it is a personal device.

---

## 6. VLAN Design

### Cisco Switch Configuration
The Cisco switch will carry all VLANs as a trunk. Proxmox nodes connect via trunk ports passing all VLANs. pfSense performs inter-VLAN routing with firewall enforcement at zone boundaries.

### Unmanaged Netgear Switch
The Netgear switch is VLAN-unaware but will pass 802.1Q tagged frames transparently. This means a VLAN trunk can run through it end-to-end between the Cisco switch and pfSense. Devices plugged directly into the Netgear (Xbox, WAN uplink) share an untagged broadcast domain at that switch — acceptable given the low sensitivity of those devices. The Cisco and pfSense enforce real segmentation downstream.

**Future improvement:** Running a second cable directly from the bedroom entry point to the Cisco switch would allow retiring the Netgear and giving pfSense full VLAN control from the first hop.

---

## 7. Physical Topology

```
[Xfinity Gateway] (east family room)
        |
    Cat6 ~40ft (crawlspace run)
        |
[Netgear Unmanaged Switch] (under desk, SW bedroom)
        |                    |
      [Xbox]         Cat6 ~30ft (perimeter run)
                             |
               [Cisco Managed Switch] (NE corner, bedroom)
                             |
              _______________+_______________
              |              |               |
       [Proxmox Node 1] [Proxmox Node 2] [Proxmox Node 3]
                                            
              [Desktop] (also on Cisco switch)

[Laptop] — wireless via Xfinity gateway
```

---

## 8. Proxmox Cluster & VM Layout

### Storage Decision
Each node uses local RAID 1 storage (500GB usable). Shared storage (Ceph) is deferred until larger drives are available — at 500GB the overhead is not justified. Live migration and HA are limited under this model but acceptable for the current build stage.

**Storage watch item:** Windows VMs in the lab consume 40-60GB each. Monitor Node 3 storage usage carefully and destroy unused VMs aggressively.

### Node Assignment Philosophy

| Node | Role | Workloads |
|---|---|---|
| Node 1 | Infrastructure | pfSense, Proxmox Backup Server, Monitoring stack |
| Node 2 | DMZ Services | Reverse proxy, web, mail, game servers, Nextcloud |
| Node 3 | Lab | Kali, AD lab, vulnerable VMs, K3s cluster |

Node assignment is soft — Proxmox allows live migration. The intent is to prevent lab workloads from competing with production services for resources.

### VM Layout

#### Node 1 — Infrastructure
| VM / LXC | Type | Zone | Purpose |
|---|---|---|---|
| pfSense | VM | All (trunk) | Firewall, routing, DHCP, DNS, VPN |
| Proxmox Backup Server | VM | Management | Incremental VM backups |
| Monitoring Stack | LXC / VM | Management | Grafana + Prometheus + Loki |
| Wazuh SIEM | VM | Management | Security event correlation |

#### Node 2 — DMZ Services
| VM / LXC | Type | Zone | Purpose |
|---|---|---|---|
| Reverse Proxy | LXC (Docker) | DMZ | Nginx Proxy Manager, TLS termination |
| DMZ App Host | VM (Docker) | DMZ | Nextcloud, personal website |
| Mail Server | VM | DMZ | Postfix/Dovecot stack |
| Game Server Host | VM (Docker) | DMZ | Minecraft, Arma 3 |

#### Node 3 — Lab
| VM / LXC | Type | Zone | Purpose |
|---|---|---|---|
| Kali Linux | VM | Lab | Primary attack machine |
| AD Domain Controller | VM | Lab | Windows Server, AD lab |
| Windows Clients (x2) | VM | Lab | AD member machines |
| Vulnerable VMs | VM | Lab | Practice targets (ephemeral) |
| K3s Node (x2-3) | VM | Lab | Kubernetes learning cluster |

#### Future — Storage
| VM / Device | Type | Zone | Purpose |
|---|---|---|---|
| TrueNAS | Dedicated hardware | Storage (VLAN 70) | NAS, SMB/NFS shares |
| Jellyfin | LXC / VM | DMZ or VPN-gated | Media server |

### LXC vs Full VM Guidance

**Use LXC containers for:**
- Reverse proxy
- Monitoring and logging stack
- Nextcloud (standard stack)
- Lightweight web services

**Use full VMs for:**
- pfSense (requires own kernel)
- All Windows machines
- Mail server
- Anything in the hacking lab (stronger isolation)

### Docker Strategy
Proxmox VMs and LXC containers define **security isolation boundaries** (VLAN, firewall). Docker runs **inside** those boundaries to manage application workloads. This gives zone-level isolation from Proxmox and application-level convenience from Docker.

pfSense and security-critical infrastructure are never run in Docker.

---

## 9. Key Technology Decisions

### Firewall / Router
**pfSense** — deployed as a VM on Node 1. Handles all inter-VLAN routing, firewall rules, DHCP per zone, DNS resolution, DDNS updates, WireGuard VPN, and Suricata IDS.

### Reverse Proxy
**Nginx Proxy Manager** (Docker) — single point of entry for all web traffic on ports 80/443. Routes by hostname to internal services. Handles Let's Encrypt TLS certificate issuance and renewal.

Game servers (Minecraft port 25565, Arma 3 port 2302 UDP etc.) bypass the reverse proxy via direct port forwarding — they use non-HTTP protocols.

### DNS
Internal DNS handled by pfSense DNS Resolver. Split-horizon DNS allows internal hostnames to resolve to internal IPs while external DNS resolves to the public IP.

### Dynamic DNS
**Cloudflare** (free tier) — pfSense DDNS client updates the domain's A record automatically when the residential IP changes.

### TLS / Certificates
- **External:** Let's Encrypt wildcard certificate (*.yourdomain.com) managed by the reverse proxy
- **Internal:** Internal CA (pfSense or Step-CA) for services not exposed externally

### VPN
**WireGuard** on pfSense — provides remote access for laptop and phone. VPN is the only on-ramp for the wireless zone to reach internal services.

### Monitoring
- **Grafana + Prometheus** — infrastructure metrics
- **Loki** — log aggregation
- **Wazuh** — SIEM, security event correlation, agent-based host monitoring

### IDS/IPS
**Suricata** — runs natively inside pfSense, inspects traffic crossing zone boundaries.

### Backup
**Proxmox Backup Server (PBS)** — VM on Node 1, incremental backups of all cluster VMs. Future: NAS as secondary backup target for 3-2-1 compliance.

### Kubernetes
**K3s** — lightweight Kubernetes for learning purposes, deployed on Node 3 in isolated lab VMs. Not used for production infrastructure.

### Personal Cloud
**Nextcloud** — deployed in DMZ, behind reverse proxy, 2FA mandatory before public exposure.

### Media Server
**Jellyfin** — deployed internally, accessed via WireGuard VPN rather than direct public exposure. Revisit if direct public access becomes necessary.

### Mail Server
**Postfix + Dovecot** — deployed in DMZ. Requires SPF, DKIM, and DMARC DNS records. Outbound relay (Mailgun or AWS SES) likely needed to avoid residential IP blocklist issues.

### NAS
**TrueNAS** on dedicated hardware (future). Assigned to Storage VLAN 70, accessible from Management and DMZ zones only.

---

## 10. Major Risks & Considerations

### CGNAT — Verify Before Building
Xfinity in many areas places residential customers behind Carrier Grade NAT. If your WAN IP (visible in the Xfinity gateway) does not match your public IP (whatismyip.com), you are behind CGNAT and port forwarding is impossible without contacting Xfinity to request a true public IP, or implementing a VPS tunnel (small VPS with public IP, WireGuard tunnel back to home).

**Action:** Verify this before any public-facing architecture is built.

### Single Public IP
One public IP handles all public-facing services. The reverse proxy is the mechanism that makes this work for web traffic. Game servers and mail require individual port forwards. Plan port assignments carefully to avoid conflicts.

### Dynamic Residential IP
Residential IPs change periodically. DDNS (Cloudflare via pfSense) must be configured before any domain-based service goes live.

### Backup — RAID Is Not Backup
RAID 1 protects against drive failure. It does not protect against accidental deletion, ransomware, VM corruption, or misconfiguration. PBS must be configured before any real data lives on this network. Long-term, a 3-2-1 backup strategy (3 copies, 2 media types, 1 offsite) is the target.

### Mail Server Complexity
Mail is the most complex public-facing service:
- Requires SPF, DKIM, DMARC DNS records configured correctly
- Residential IPs are frequently on blocklists — outbound mail may be rejected by Gmail/Outlook regardless of configuration
- Outbound relay service (Mailgun, AWS SES) likely required
- Do not treat as primary personal email until deliverability is fully validated

### Hacking Lab Isolation — Non-Negotiable
The lab VLAN must have zero lateral access to any other zone. Outbound internet access from the lab should be blocked or strictly whitelisted. Intentionally vulnerable machines on an under-segmented network represent serious risk. The lab is built last, after all other segmentation is validated.

### Management Plane Security
pfSense and Proxmox admin interfaces must be:
- Accessible only from Management VLAN
- Never exposed to the internet
- Protected with strong credentials and 2FA
- Covered by centralized logging

### pfSense Bootstrap Risk
pfSense will manage the network that Proxmox itself lives on. Careful sequencing during initial setup is required to avoid locking yourself out of cluster management. Plan the cutover from Xfinity gateway routing to pfSense routing deliberately.

### Storage Constraints
500GB per node fills up faster than expected with Windows VMs (40-60GB each). Node 3 (lab) is most at risk. Destroy unused VMs aggressively and consider drive upgrades when budget allows. Ceph (shared storage) should be reconsidered when drives are larger.

### Windows Licensing
AD lab requires Windows Server and Windows client licenses. Microsoft evaluation ISOs (180-day trial) are available free and are appropriate for lab use.

---

## 11. Build Order

*NOTE: Items marked with 'D' have been deferred until later.*
*NOTE: Items marked with 'X' have been completed.*

### Phase 0 — Verify Assumptions
*Complete before touching anything*

- [X] Verify CGNAT status — compare Xfinity gateway WAN IP vs whatismyip.com
        Note: Xfinity does not use CGNAT
        Note: House ip is 68.50.74.90
- [X] Audit Cisco switch — confirm model, VLAN support, IOS version
        Note: Switch version is sg200-s. This is capable of VLAN support
- [X] Document current state — physical and logical diagrams of existing network
        Note: I have created a logical diagram of the current infrastructure.
        Note: I have created a json file detailing connection methods for all relevant devices.
- [D] Purchase a domain name

### Phase 1 — Core Infrastructure
*Build the network skeleton. Nothing public-facing yet.*

- [X] Deploy pfSense VM on Node 1
- [X] Configure VLANs on Cisco switch (trunking, port assignments)
- [X] Validate inter-VLAN routing and zone firewall rules in pfSense
- [X] Migrate desktop to Management VLAN
- [X] Lock Proxmox management interface to Management VLAN only
- [D] Configure DDNS on pfSense (Cloudflare) 

**Milestone:** Network is segmented, management plane is protected, firewall zones are enforced.

### Phase 2 — Observability
*Visibility before exposure.*

- [X] Deploy Suricata on Pfsense
- [X] Deploy and connect metric/log console - Grafana
- [X] Deploy centralized metric aggregator — Prometheus
- [X] Ship pfsense metrics to central metric machine
- [X] Ship proxmox metrics to central metric machine
- [X] Ship cisco metrics to central metric machine
- [X] Deploy centralized log aggregator — Loki 
- [X] Ship pfSense logs to central logging
- [X] Ship Proxmox logs to central logging
- [X] Ship cisco logs to central logging
- [X] Deploy Wazuh SIEM
- [D] Deploy Proxmox Backup Server VM
- [D] Configure backup jobs for all infrastructure VMs

**Milestone:** Full visibility across the network. Infrastructure VMs are being backed up.

### Phase 3 — DMZ Foundation
*Scaffold the DMZ before adding services.*

- [ ] Deploy reverse proxy VM/LXC in DMZ (Nginx Proxy Manager in Docker)
- [ ] Configure internal CA for internal TLS
- [ ] Validate port forwarding — 80/443 WAN to reverse proxy only
- [ ] Configure Let's Encrypt wildcard certificate on reverse proxy
- [ ] Deploy a static test page to validate the full request path end-to-end

**Milestone:** A request from the internet reaches the reverse proxy and receives a response. The full public-facing pipeline is proven.

### Phase 4 — Public Services
*One service at a time. Validate each before adding the next.*

- [ ] Minecraft server — direct port forward, validate DMZ isolation
- [ ] Arma 3 server — same pattern
- [ ] Personal website — behind reverse proxy
- [ ] Nextcloud — behind reverse proxy, 2FA enabled before exposure
- [ ] Jellyfin — deploy internally first, evaluate VPN-gated vs public access

### Phase 5 — Mail Server
*Treated as its own project.*

- [ ] Research and resolve residential IP blocklist situation
- [ ] Deploy mail server VM in DMZ
- [ ] Configure Postfix/Dovecot
- [ ] Configure SPF, DKIM, DMARC DNS records
- [ ] Configure outbound relay if needed
- [ ] Test deliverability thoroughly before treating as production

### Phase 6 — Hacking Lab
*Only after all other network segmentation is stable and validated.*

- [ ] Create Lab VLAN with strict outbound firewall rules
- [ ] Deploy Kali Linux VM — bridge to Lab VLAN when in use
- [ ] Build AD lab — Domain Controller + Windows clients
- [ ] Implement Game of Active Directory
- [ ] Spin up vulnerable VMs as needed (ephemeral)
- [ ] Deploy K3s cluster (2-3 VMs) for Kubernetes learning

### Phase 7 — Storage
*When dedicated hardware is available.*

- [ ] Deploy TrueNAS on dedicated hardware
- [ ] Assign to Storage VLAN 70
- [ ] Configure SMB/NFS shares accessible from Management VLAN
- [ ] Integrate with Nextcloud as expanded storage backend
- [ ] Add NAS as secondary Proxmox Backup Server target

---

## 12. Documentation Standards

Running this project as a portfolio piece means documentation is as important as the technical build itself.

### Maintain Throughout the Project

- **Network diagrams** — logical (VLANs, zones, traffic flows) and physical (cables, hardware locations) maintained separately. Recommended tool: draw.io
- **Change log** — date, what changed, why. Every meaningful configuration change.
- **Runbooks** — per service: how to deploy it, how to restore it from backup, how to troubleshoot common failures
- **Threat model** — a living document describing what assets exist, what threats are relevant, and what controls are in place

### Proxmox Hygiene

- Snapshot before every major change
- Descriptive VM and snapshot names (include date)
- Destroy lab VMs that are no longer in use — don't let Node 3 fill up with abandoned machines

### A Note on the Portfolio Angle

When presenting this project professionally, the most valuable things to be able to demonstrate are:

1. The reasoning behind architectural decisions, not just the decisions themselves
2. How you handled something that went wrong
3. The monitoring and logging stack — this shows security maturity beyond just "I set up servers"
4. The threat model — this shows you thought about what you were protecting and from whom
5. The documentation itself — organized, current, and readable

---

## 13. Note From the Human

This document is the product of a 3 hour long conversation I had with Claude. While I understand a decent amount of networking stuff on a theoretical level, I am a bit inexperienced with larger environments. Hence my undertaking of this project. Claude fills the role of a knowledgeable entity (something like a professor or a baby network architect) with which I can have a constructive dialog. Such a role would otherwise be left to endlessly sifting through ancient Reddit threads, vague Youtube rabbit holes, or simply missed by ignorance. These are rather unsavory options.

Additionally, Claude has acted as a "second pair of eyes" from which top approach my projects. While there are some clear objectives I may have for a given project, discussions with Claude allow me to see holes in my plan. For instance, this entire homelab project started with my hope to create a significant, stable, and security environment from which I could run public servers and internal labs. Some things I knew I should have, such as a reverse proxy. However, I had not considered the necessity of items like and IDS or a SIEM. Not to say that those are 100% necessary, but, as I want this build to reflect a higher caliber, these additions are certainly a positive

My use of Claude through out this project is to act as a sounding board and tutor for this build. I would love to be able to have a face-to-face with experienced professionals or scroll message boards to learn everything, but, realistically speaking, this isn't pragmatic. Insofar as I take advantage of AI tools, it is my intention to examine, question, and truly understand the concepts and techniques presented to me. My favorite thing about AI tools like Claude and Gemini is that there is a two-way street of communication, however artificial. If I have questions or comments I can raise these in conversations. The AI tool is free to push back and defend it's assertion or affirm my comments to the best of its understanding. This is not something I can readily get from search engines or chat forums. 

AI can never and should never replace human creativity. However, in technical and best practice instances, I have found it to be a great service. Still, I want to be transparent and honest about my use of AI in public projects such as this. I hope to never become one of those individuals who can't think or do anything because of their reliance on AI. Instead I hope to know every in and out of the suggestions AI gives me. I hope to learn from my conversations so that, without AI, I am still capable and competent.

-BV

## Disclaimer

This document was generated by an instance of Claude Sonnet 4.6. It was reviewed and modified as necessary. It's authenticity and accuracy to the content therein has been verified and approved.

-BV