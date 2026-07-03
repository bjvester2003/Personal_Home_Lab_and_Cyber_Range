# Home Network & Lab — Phase 2 Completion Report

## Overview

Phase 2 builds observability and security monitoring on top of the Phase 1
network foundation. The goal was full visibility across the existing
infrastructure (pfSense, the Proxmox cluster, and the Cisco switch) via
centralized logging and metrics, plus active security analysis via a
self-hosted SIEM and an inline IDS. All five planned tools — Loki,
Prometheus, Grafana, Wazuh, and Suricata — are deployed and ingesting real
data from every Phase 1 device by the end of this phase.

---

## Objectives (from objectives.md, Phase 2 scope)

- Deploy Suricata on Pfsense
- Deploy and connect metric/log console - Grafana
- Deploy centralized metric aggregator — Prometheus
- Ship pfsense metrics to central metric machine
- Ship proxmox metrics to central metric machine
- Ship cisco metrics to central metric machine
- Deploy centralized log aggregator — Loki 
- Ship pfSense logs to central logging
- Ship Proxmox logs to central logging
- Ship cisco logs to central logging
- Deploy Wazuh SIEM
- Proxmox Backup Server VM + backup jobs — **deferred**, pushed to a later
  phase per operator decision; not attempted in Phase 2

Two items were added to scope organically during the phase, beyond the
original objectives list: Cisco switch integration (metrics via SNMP, logs
via syslog) and Wazuh agent deployment to endpoints, both judged necessary
for the stack to be genuinely comprehensive rather than pfSense/Proxmox-only.

---

## Architecture / Pipeline Map

```
                         ┌────────────────────────────────────────────┐
                         │              GRAFANA (10.10.10.13)         │
                         │     dashboards — queries Loki + Prometheus │
                         └──────────────┬─────────────┬───────────────┘
                                        │             │
                          ┌─────────────┘             └─────────────┐
                          ▼                                         ▼
            ┌──────────────────────────┐                ┌──────────────────────────┐
            │   LOKI (10.10.10.11)     │                │ PROMETHEUS (10.10.10.12) │
            │   log storage, :3100     │                │ metrics store, :9090     │
            └─────────────▲────────────┘                └──────────────▲───────────┘
                          │                                            │
        ┌─────────────────┼─────────────────┐          ┌───────────────┼───────────────┐
        │                 │                 │          │               │               │
   Alloy (local)     Alloy (local)    rsyslog→Alloy   node_exporter  pfSense exporter  snmp_exporter
   syslog :1514       journal          :1614→:1515     :9100 (x3)      :9100            :9116
   (pfSense,           shipper          (Cisco SG200,   on each         on pfSense       on Prometheus
   native RFC5424)     on each          RFC3164→RFC5424 Proxmox node                     host, queries
                        Proxmox node     normalized)                                      switch via
                                                                                            SNMP UDP 161
        │                 │                 │
   pfSense           singledecker      Cisco SG200-26
   (10.10.10.1)       doubledecker      (10.10.10.2)
                       tripledecker

            ┌────────────────────────────────────────────────────┐
            │              WAZUH (10.10.10.14)                   │
            │  manager + indexer + dashboard, https :443         │
            │  agent comms :1514/:1515, syslog collector :514/udp│
            └──────┬──────────┬──────────┬──────────┬────────────┘
                   │          │          │          │
              Wazuh agent  Wazuh agent  Wazuh agent  Wazuh agent
              singledecker doubledecker tripledecker  desktop

            (Wazuh also receives syslog directly from pfSense,
             Cisco switch, and Suricata alerts via :514/udp,
             duplicating what Loki receives for security analysis)

   Suricata (on pfSense, WAN interface) → alerts via syslog → Loki + Wazuh
```

---

## Devices & Hosts

| Host | Type | Node | IP | Role |
|---|---|---|---|---|
| Loki | LXC (101) | doubledecker | 10.10.10.11 | Log storage |
| Prometheus | LXC (102) | doubledecker | 10.10.10.12 | Metrics storage |
| Grafana | LXC (103) | doubledecker | 10.10.10.13 | Dashboards / visualization |
| Wazuh | VM (104) | doubledecker | 10.10.10.14 | SIEM (manager + indexer + dashboard) |
| pfSense | VM (100) | singledecker | 10.10.10.1 | Firewall/router (unchanged from Phase 1) |
| singledecker | Proxmox node | — | 10.10.10.6 | Proxmox node 1 / pfSense host |
| doubledecker | Proxmox node | — | 10.10.10.7 | Proxmox node 2 / Phase 2 stack host |
| tripledecker | Proxmox node | — | 10.10.10.8 | Proxmox node 3 / reserved for lab |
| Cisco SG200-26 | Switch | — | 10.10.10.2 | Managed switch (unchanged from Phase 1) |

All Phase 2 hosts are on VLAN 10 (MGMT), consistent with the Phase 1
decision to keep observability infrastructure on the management network for
visibility reasons.

---

## Resource Allocation

| Service | Host | vCPU | RAM | Disk | Type |
|---|---|---|---|---|---|
| pfSense + Suricata | singledecker | 4 | 6–8GB | — | VM |
| Loki | doubledecker | 1 | 1GB | 8GB | LXC |
| Prometheus | doubledecker | 1 | 1GB | 8GB | LXC |
| Grafana | doubledecker | 1 | 1GB | 8GB | LXC |
| Wazuh | doubledecker | 4 | 8GB | 40–60GB | VM |

**Host-installed software (bare Proxmox OS, not in a guest):** `node_exporter`
(all 3 nodes), Grafana Alloy journal shipper (all 3 nodes), Wazuh agent (all
3 nodes). Flagged here deliberately — three separate pieces of third-party
software are now running directly on the hypervisor hosts rather than in
disposable guests. Each individually is small and well-scoped (read-only
metrics/log exporters, one security agent), but the accumulation is worth
tracking.

---

## Firewall Changes (pfSense)

### New Aliases

| Alias | Type | Value(s) | Purpose |
|---|---|---|---|
| `monitoring_and_security` | Host(s) | 10.10.10.11, .12, .13, .14 | Phase 2 monitoring/SIEM stack |

### New/Modified Rules — MGMT

| Rule | Source | Destination | Port/Protocol | Purpose |
|---|---|---|---|---|
| Phase 2 infra internet access | `monitoring_and_security` | `!RFC1918` | any | Package updates, threat intel feeds |
| pfSense syslog to Loki | `pfsense` | `monitoring_and_security` | UDP/1514 | Firewall/system log shipping |
| Cisco syslog to Loki | `cisco_switch` | `monitoring_and_security` | UDP/1614 | Switch log shipping |
| Prometheus → pfSense exporter | `monitoring_and_security` | `pfsense` | TCP/9100 | pfSense metrics scrape |
| Monitored hosts → monitoring stack | (broad, see note) | `monitoring_and_security` | any | **Temporary** — see Known Gaps |

**Note on the broad rule:** during buildout, a wide-open "monitored devices →
monitoring_and_security: any" rule was added to unblock progress while connecting
multiple sources at once. This was a deliberate, acknowledged shortcut, not
an oversight — it is explicitly flagged for tightening into least-privilege,
per-service rules in a follow-up session (see Known Gaps).

---

## Services & Ports Reference

| Service | Port | Protocol | Notes |
|---|---|---|---|
| Loki HTTP API | 3100 | TCP | Push + query |
| Loki gRPC | 9096 | TCP | Internal, unused at single-binary scale |
| Alloy syslog listener (pfSense) | 1514 | UDP | Native RFC5424 from pfSense |
| Alloy syslog listener (Cisco) | 1515 | UDP | Localhost-only, receives from rsyslog normalizer |
| rsyslog intake (Cisco) | 1614 | UDP | Raw RFC3164 from switch, normalized to RFC5424 |
| Prometheus web/API | 9090 | TCP | |
| node_exporter | 9100 | TCP | Per Proxmox node, and pfSense's exporter package |
| snmp_exporter | 9116 | TCP | Runs on Prometheus host, polls switch via SNMP |
| SNMP (switch) | 161 | UDP | Read-only community, source-restricted |
| Grafana web UI | 3000 | TCP | |
| Wazuh dashboard | 443 | TCP (HTTPS) | Self-signed cert by default |
| Wazuh agent events | 1514 | TCP | Coincidentally same number as Loki's syslog port, different host — no actual conflict |
| Wazuh agent enrollment | 1515 | TCP | |
| Wazuh syslog collector | 514 | UDP | Receives pfSense, Cisco, Suricata alerts |

---

## Technology Decisions & Rationale

Full comparative research (Loki vs. ELK/Graylog, Prometheus vs.
Zabbix/Influx, Grafana vs. Kibana, Wazuh vs. Splunk/Elastic Security,
Suricata vs. Snort) is documented separately in `phase2-tool-research.md`,
compiled during planning before deployment began. Summary of decisions:

- **Loki** chosen over ELK/Graylog for lightweight resource use at homelab
  log volume and native Grafana integration; tradeoff is weaker free-text
  search, acceptable at this scale.
- **Prometheus + Grafana** chosen as the de facto open-source standard for
  this role; effectively uncontested at this scale.
- **Wazuh** chosen over Splunk specifically because Splunk's free tier
  ingest cap (500MB/day) is impractical for genuine self-hosting; Wazuh is
  free, fully self-hosted, and well-recognized in security/blue-team
  circles for portfolio purposes.
- **Suricata** chosen over Snort despite Snort's lighter single-core CPU
  footprint, because Suricata's EVE JSON output integrates far more
  cleanly with the logging-centric architecture than Snort's older
  alert formats.
- **Tempo (LGTM stack)** evaluated and deliberately **not** deployed —
  distributed tracing has no value without multiple instrumented services
  for a request to traverse, which doesn't exist yet in this environment.
  Revisit once an actual multi-component application (e.g. behind the
  future reverse proxy) is built and instrumented.

---

## Issues Encountered & Resolutions

### 1. No LXC templates available on doubledecker
**Symptom:** Create CT wizard showed no templates to select.
**Cause:** Proxmox doesn't ship templates pre-downloaded; they must be
pulled from the online repository first.
**Fix:** `pveam update && pveam download local debian-12-standard_<ver>_amd64.tar.zst`.

### 2. Systemd 252 nesting warning, container failed to start
**Symptom:** `WARN: Systemd 252 detected. You may need to enable nesting.`
**Cause:** Modern systemd in current templates wants the nesting feature
enabled even without Docker involved.
**Fix:** `pct set <vmid> -features nesting=1`, then `pct start <vmid>`.

### 3. Network device creation failure on container start
**Symptom:** `lxc_create_network_priv: Failed to create network device`.
**Cause:** IPv6 was set to **Static** in the network device dialog with
blank CIDR/Gateway fields — Proxmox attempted to apply an invalid static
config rather than ignoring the empty fields.
**Fix:** Changed IPv6 to **DHCP** (no usable "None" option in this Proxmox
version); since no IPv6 DHCP server exists on VLAN 10, it fails silently
and harmlessly, equivalent to disabled.

### 4. Container created but couldn't be pinged; doubledecker's vmbr0 not VLAN-aware
**Cause:** Setting a VLAN tag on the container's `net0` requires the bridge
itself to be VLAN-aware in order to tag traffic. This matched a gap already
flagged in the Phase 1 report: doubledecker's `vmbr0` was never made
VLAN-aware (only singledecker's was, after Phase 1 Issue #6).
**Fix:** Enabled **VLAN aware** on doubledecker's `vmbr0` (System →
Network → edit vmbr0), required a node networking reload/reboot to apply.

### 5. Container reachable from desktop, but container couldn't ping its own gateway or internet
**Cause:** Two layers, diagnosed separately:
  - Proxmox's own per-guest firewall (enabled via the `firewall=1` checkbox
    on `net0`) defaults to drop-all when no rules exist — separate from and
    in addition to pfSense's firewall.
  - Desktop→container worked because desktop is the *source* and covered
    by pfSense's MGMT rule #1; container→gateway/internet failed because
    the container, as *source*, wasn't covered by any pfSense alias and
    hit the MGMT default-deny rule — including for traffic addressed to
    pfSense's own interface IP, which is still evaluated by that
    interface's ruleset even though no VLAN crossing occurs.
**Fix:** Disabled Proxmox's per-guest firewall (relying on pfSense alone,
single source of truth). Created the `monitoring_and_security` alias and a pass rule
(`monitoring_and_security → !RFC1918`) on pfSense's MGMT tab.

### 6. New pfSense pass rule added but traffic still blocked
**Cause:** New rules are appended to the bottom of the rule list by
default; the rule landed below the default-deny catch-all, so the
catch-all matched first (pfSense evaluates top-down, first match wins).
**Fix:** Manually dragged the rule above the default-deny rule. Separately
discovered that dragging alone doesn't persist the new order — an explicit
**Save** is required after reordering, distinct from Apply Changes.

### 7. Re-enabling vmbr0 VLAN-awareness broke doubledecker's own MGMT connectivity
**Cause:** Same root-cause class as Phase 1 Issue #6: doubledecker's
existing `vlan10` management interface was tapped directly off the raw NIC
(`nic0`), not off the bridge. Once `vmbr0` became VLAN-aware, it began
competing with the NIC-tapped `vlan10` device for the same tagged frames.
**Fix:** Deleted the NIC-tapped `vlan10` interface, recreated it tapped off
`vmbr0` instead — identical fix to Phase 1 Issue #6, applied to the second
node. **Noted for follow-up:** the same fix will be needed on tripledecker
once anything is deployed there.

### 8. Promtail binary missing from current Loki release assets
**Cause:** Promtail reached end-of-life in March 2026; Grafana removed it
from current release packaging, with all future development moved to
Grafana Alloy.
**Fix:** Switched the log-shipping agent to Grafana Alloy for the entire
project rather than installing an EOL tool. No prior Promtail deployment
existed to migrate, so this was a clean start on Alloy throughout.

### 9. Alloy syslog parser rejecting all pfSense messages
**Symptom:** `error parsing syslog stream... expecting a version value in
the range 1-999`.
**Cause:** pfSense's default syslog output is BSD-style (RFC3164), which
lacks a version field; Alloy's syslog receiver only understands RFC5424.
This is a known, unsupported gap upstream (BSD syslog over UDP was never
built into Loki's/Alloy's syslog ingester).
**Initial plan:** stand up rsyslog as a format-normalizing relay
(RFC3164 → RFC5424) in front of Alloy.
**Actual fix:** discovered pfSense has a native **Log Message Format: RFC
5424** setting (Status → System Logs → Settings), eliminating the need for
a normalizer for pfSense specifically. Alloy was pointed back at port 1514
directly; rsyslog was disabled (not removed) for later reuse.

### 10. Port 1514 conflict when reverting Alloy to listen there directly
**Cause:** rsyslog, stopped via `systemctl stop`, did not fully release
the port — the process had not actually terminated.
**Fix:** `pkill -9 rsyslogd`, confirmed via `ps aux` no process remained,
then started cleanly.

### 11. Prometheus failed to start after adding the pfSense scrape job (exit code 2)
**Cause:** YAML indentation mismatch in the new job block under
`scrape_configs` — a stray/missing space, since YAML is whitespace-sensitive.
**Fix:** Corrected indentation to match the existing job blocks exactly.

### 12. Prometheus 3.x tarball missing `consoles`/`console_libraries` directories
**Cause:** Prometheus 3.x removed the legacy console-template web UI;
training-data-era install instructions assumed an older release structure.
**Fix:** Skipped the `mv` steps for those directories entirely — not
required by the current version, and Grafana is the real visualization
layer regardless.

### 13. SNMP exporter returning HTTP 500 / "connection refused" from the switch
**Cause:** "Connection refused" (an active ICMP port-unreachable, as
opposed to a timeout) indicated nothing was listening on UDP 161 at all —
not an auth/community-string problem as initially suspected. The SG200-26's
SNMP agent required an explicit service-level enable under **Security →
TCP/UDP Services**, separate from configuring the SNMP Engine ID and
Community pages.
**Fix:** Enabled the SNMP service under the Security/TCP-UDP Services tab.

### 14. Cisco syslog never arriving at Loki — duplicate rsyslog processes
**Cause:** An earlier `systemctl stop rsyslog` (from Issue #9/10's cleanup)
had not fully terminated the process; restarting later for the Cisco
pipeline produced a second concurrent rsyslogd instance, causing module/port
conflicts and unreliable forwarding.
**Fix:** Same as Issue #10 — `pkill -9 rsyslogd`, confirmed single clean
instance before reconfiguring.

### 15. Cisco syslog forwarding rule silently using TCP instead of UDP
**Cause:** A typo in the rsyslog forwarding action — `@@127.0.0.1:1515`
(double-`@`, rsyslog's TCP-forward syntax) instead of the intended single
`@` for UDP — caused rsyslog to attempt a TCP connection against Alloy's
UDP-only listener, producing persistent "connection refused" errors.
**Fix:** Corrected to single `@` for UDP forwarding.

### 16. Wazuh agent failed to start on singledecker — literal placeholder in config
**Symptom:** `ERROR: Invalid server address found: 'MANAGER_IP'`.
**Cause:** The literal placeholder string `MANAGER_IP` ended up in
`/var/ossec/etc/ossec.conf` instead of the actual Wazuh manager IP,
likely from not using the dashboard's exact auto-generated install command.
**Fix:** Manually edited `<address>MANAGER_IP</address>` to
`<address>10.10.10.14</address>` in `ossec.conf`, restarted the agent.

### 17. Wazuh archives.log empty despite a confirmed-working syslog listener
**Cause:** `logall` and `logall_json` are disabled by default in
`ossec.conf` — Wazuh only writes the full raw-event archive when explicitly
told to log everything, not just rule-matched alerts. Traffic was arriving
correctly; nothing was being written to disk to show for it.
**Fix:** Enabled `<logall>yes</logall>` and `<logall_json>yes</logall_json>`
under `<global>`, restarted `wazuh-manager`.

---

## Known Gaps / Deferred Items

| Item | Priority | Notes |
|---|---|---|
| Grafana dashboards | High | Only the imported community "Node Exporter Full" (ID 1860) exists; no custom panels for pfSense, Wazuh, Loki log views, or the switch yet |
| pfSense/Suricata log parsing (`loki.process`) | Medium | Logs currently stored as raw unparsed strings (pfSense's CSV filterlog format, Suricata EVE-via-syslog); no structured labels for action/protocol/port filtering yet |
| Firewall rule tightening | High | "Monitored hosts → monitoring_and_security: any" rule is intentionally broad for buildout speed; needs conversion to least-privilege, per-service/port rules |
| DMZ → MGMT logging exception | Deferred to Phase 3 | DMZ web/game servers will eventually need to ship logs into Loki, which conflicts with the Phase 1 "DMZ → Management: Deny all" rule. Decision made to implement a narrow, explicitly-documented single-host/single-port exception when DMZ servers actually exist, rather than now |
| Cisco switch SNMP — generic IF-MIB only | Low | Using `snmp_exporter`'s default IF-MIB module (interface counters/status); no Cisco-specific OIDs (fan, temp, etc.) configured |
| Wazuh dashboard/query proficiency | Low | Operator can confirm ingestion via `archives.log` but has not yet learned the dashboard's OpenSearch-based query/filtering UI |
| Proxmox Backup Server | Medium | Explicitly pushed back from Phase 2 scope per operator decision; not attempted |
| Tempo / full LGTM stack | Low | Evaluated and deliberately deferred — no multi-hop instrumented application exists yet to make tracing useful |
| Host-installed third-party software accumulation | Medium | `node_exporter`, Alloy (journal shipper), and Wazuh agent now run directly on all three bare Proxmox hosts; worth a periodic audit as more is added |
| Single point of failure — doubledecker | Long-term | Loki, Prometheus, Grafana, and Wazuh all live on one physical node; loss of doubledecker means total loss of both logging and security monitoring simultaneously. Accepted risk pending budget for real HA/clustering (compounds the existing Phase 1 "pfSense as a VM" risk in a different way) |


---

## Phase 3 Preview

Phase 3 is not yet fully scoped, but anticipated work includes:

- Building out real Grafana dashboards across all data sources
- Implementing the deferred `loki.process` parsing for structured log fields
- Tightening the broad Phase 2 firewall rules into least-privilege rules
- DMZ services: web/game servers, reverse proxy — and the associated narrow,
  documented logging exception to the DMZ→MGMT deny-all rule
- Proxmox Backup Server implementation (carried over from Phase 2)
- Revisiting Cisco vendor-specific SNMP MIBs and the Tempo/tracing question
  once there's an actual application to monitor
- Long-term: HA/clustering once budget allows, addressing the single-point-
  of-failure risk on doubledecker (and pfSense-on-VM from Phase 1)

---

## Disclaimer

This document was generated by an instance of Claude Sonnet 4.6, working
alongside the operator through the Phase 2 build and troubleshooting
process described above. It was reviewed and modified as necessary. Its
authenticity and accuracy to the content therein has been verified and
approved.

-BV
