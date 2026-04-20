# Practical Network Hardening

*A real-world network hardening project for a small business*

## Overview

This repository documents the end-to-end hardening of a small business network, from an unstructured flat network with tangled cabling to a properly segmented, firewalled, and monitored infrastructure.

The project is deliberately pragmatic: it uses existing hardware where possible, open-source software, and accepts documented trade-offs typical of small business environments (limited budget, no dedicated IT staff, continuity requirements for phone lines and daily operations).

Status: In progress. Physical layer and planning complete; virtualization and segmentation in implementation phase.

Reference: Loosely based on *Cybersecurity for Small Networks* by Seth Enoka, adapted to the specific constraints of this environment.

---

## Why This Project

The business network started as a flat, unsegmented LAN where every device — office workstations, printers, CCTV, IoT devices, guest phones — shared the same broadcast domain with no firewall rules between them. Cabling was disorganized, with legacy telephone wiring mixed with Ethernet and no organization of the physical equipment.

The goal is to bring the network up to modern small-business security standards: proper physical layout, logical segmentation, firewalling between segments, and a secured wireless infrastructure.

---

## Security Note on Documentation

For security reasons, this repository does not include the full network map, real IP assignments, device inventory, or any configuration files containing sensitive information. Publishing such details would expose the network to unnecessary risk.

What is published here is the methodology, architecture, and reasoning behind each decision — enough to demonstrate the approach without compromising the network itself.

---

## Hardware Inventory

| Component | Role |
|-----------|------|
| Mini PC (Ryzen 7000 series, 16GB RAM, 2x LAN) | Proxmox host — runs pfSense VM and web server VM |
| Netgear GS308E | Managed switch (802.1Q VLANs) |
| ISP router (FTTH, provided by ISP) | Repurposed as VoIP gateway only |
| ONT (provided by fiber operator) | Untouched — fiber-to-Ethernet conversion |

---

## Completed Phases

### Phase 0 — Network Mapping
Mapped all devices currently connected to the network, recorded MAC addresses, IPs, hostnames, and physical location. Map kept offline for security reasons.

### Phase 1 — Physical Cabling Survey
Traced every conduit running to workstations across the office to understand the existing cabling topology. Those info were saved in an conduit map for future maintenance

### Phase 2 — Cable Cleanup
Removed legacy and unused cabling (old telephone twisted pairs, disconnected runs) to reduce clutter and eliminate unknown/unmanaged copper.

### Phase 3 — Cable Re-termination
Re-terminated Ethernet cables that had been punched down with inconsistent pinouts, standardizing the entire office on a single wiring scheme (T568B).

### Phase 4 — Router Relocation
Moved the main router from under a desk (with cables draped across the floor) to a proper enclosure.

### Phase 5 — Network Cabinet
Consolidated all networking equipment (router, switch, patch panel) into a dedicated cabinet, alongside a security monitor and storage.

### Phase 6 — Cat6 Upgrade
Pulled new Cat6 Ethernet runs to replace older, lower-category cabling across the office.

---

## Planned Architecture
Internet
   │
   ▼
ONT (Open Fiber — untouched)
   │
   ▼
Mini PC (Proxmox)
 ├─ VM: pfSense (firewall/router)
 │    WAN on eth0, LAN on eth1 (802.1Q trunk)
 └─ VM: Web server (DMZ — internal jobsite app)
   │
   ▼ (trunk)
Managed switch (802.1Q)
   │
   ├─ VLAN 10 — Primary (office workstations, mgmt)
   ├─ VLAN 20 — Secondary (printer, CCTV, IoT, VoIP gateway)
   ├─ VLAN 30 — Guest (isolated Wi-Fi for clients/staff personal devices)
   └─ VLAN 40 — DMZ (web server, reachable from internet)

### VLAN Design

| VLAN | Name | Purpose |
|------|------|---------|
| 10 | Primary | Devices handling business data. Full management rights

| 20 | Secondary | Less trusted endpoints: printer, CCTV, IoT devices, VoIP gateway. No access to Primary data. |
| 30 | Guest | Internet-only network for personal devices. Fully isolated. |
| 40 | DMZ | Publicly accessible web server. Cannot reach internal segments. |

### Firewall Logic

| From → To | Policy |
|-----------|--------|
| Primary → any | Allowed (management) |
| Secondary → Primary | Denied |
| Secondary → Internet | Allowed (includes VoIP traffic) |
| Guest → Internet | Allowed only |
| Guest → any internal | Denied |
| DMZ → Internet | Allowed (outbound only) |
| DMZ → any internal | Denied |
| Internet → DMZ | Allowed on 80/443 only |

One-way exceptions (Primary can initiate, Secondary cannot) are used for shared resources like the printer and CCTV, preventing compromised Secondary devices from pivoting into the Primary segment.

---

## Design Trade-offs

This architecture runs the firewall as a VM alongside a web server on the same physical host. This is not the ideal enterprise pattern (dedicated firewall hardware), and the trade-offs are explicitly accepted:

- Single point of failure: host failure takes down both firewall and server.
- Resource contention: under load, the server could compete with the firewall for CPU/RAM.
- Hypervisor attack surface: a Proxmox vulnerability could theoretically allow VM escape.

These are considered acceptable for a small business context given budget constraints. Mitigation path (if/when justified): migrate the firewall to a dedicated low-cost mini PC, keeping the current host as application server only.

---

## Roadmap

- [ ] Install Proxmox on the mini PC
- [ ] Deploy and configure pfSense VM
- [ ] Migrate switch from port-based to 802.1Q VLANs
- [ ] Implement inter-VLAN firewall rules
- [ ] Stage device migration into VLANs
- [ ] Configure ISP router as VoIP-only client in VLAN 20
- [ ] Deploy web server VM in DMZ
- [ ] Wireless hardening (WPA3, client isolation on Guest, separate SSIDs per VLAN)
- [ ] Monitoring and detection (evaluation phase — IDS/IPS, logging, network TAP)
- [ ] VPN for remote management (evaluation phase)
