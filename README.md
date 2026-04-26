# Practical Network Hardening
### A Small Business Security Architecture — From Flat LAN to Internet-facing HTTPS App

Built solo over ~3 weeks. Reference: *Cybersecurity For Small Networks* (Seth Enoka), adapted to a real production environment with real users, real VoIP, real CCTV, real consequences if something breaks

## TL;DR for recruiters
**The problem**: A small landscaping company (~10 employees, mixed IT estate: office PCs, IoT cameras, IP printer, VoIP phones, vehicle-management web app) had everything on one flat subnet behind a TIM consumer router. Single password Wi-Fi. No segmentation. No backups. No documentation.

**What I built.**
- 5-VLAN segmented network (Management / Primary / IoT-Secondary / Guest / DMZ) on OPNsense + 802.1Q-managed switch
- DMZ-isolated VM running a Node.js web application, exposed publicly via HTTPS
- Defense-in-depth hardening across network, host, and application layers
- Automated encrypted backups to off-site cloud with tiered retention and Telegram alerting
- Public domain with auto-renewing TLS certificate, dynamic DNS resilient to ISP IP changes
- Multi-SSID Wi-Fi mapped to VLANs (office / IoT / guest)

## Architecture overview
```
                                    Internet
                                       │
                                       ▼
                          ┌─────────────────────────┐
                          │   TIM HUB+ (CPE)        │
                          │   Public IP (dynamic)   │
                          │   VoIP / DMZ Host       │
                          └────────────┬────────────┘
                                       │
                                       ▼
                          ┌─────────────────────────┐
                          │   OPNsense (VM)         │
                          │   Stateful firewall     │
                          │   DHCP / DNS / DDNS     │
                          └────────────┬────────────┘
                                       │
                          ┌────────────┴────────────┐
                          │  Netgear GS308E (L2)    │
                          │  802.1Q trunk port 6    │
                          └─┬───┬───┬───┬───┬───┬───┘
                            │   │   │   │   │   │
                ┌───────────┘   │   │   │   │   └────────────┐
                ▼               ▼   ▼   ▼   ▼                ▼
         ┌──────────┐    ┌──────────────────────────┐    ┌─────────┐
         │ VLAN 1   │    │ VLAN 10  20  30          │    │ VLAN 40 │
         │ Mgmt     │    │ Office IoT Guest         │    │ DMZ     │
         │ Proxmox  │    │ PCs    Cam WiFi          │    │ webapp  │
         │ + admin  │    │ Print  CCTV              │    │ HTTPS   │
         └──────────┘    └──────────────────────────┘    └─────────┘
```
| VLAN | ID | Subnet | Purpose | Inter-VLAN |
|---|---|---|---|---|
| Management | 1 | 192.168.10.0/24 | Proxmox, OPNsense GUI, admin laptop | Full access |
| Primary | 10 | 192.168.110.0/24 | Office PCs, IP printer | Restricted |
| Secondary | 20 | 192.168.120.0/24 | IoT, CCTV, robot mower | Internet only |
| Guest | 30 | 192.168.130.0/24 | Visitors Wi-Fi | Internet only |
| DMZ | 40 | 192.168.140.0/24 | Web application VM | Internet only — no inbound to internal |

---

## Hardware and software inventory
 
**Hardware**
- MINISFORUM AM06PRO (Ryzen 7000, 16 GB, 2× LAN) — Proxmox host
- Netgear GS308E — managed L2 switch with 802.1Q
- TIM HUB+ DN8245X6-8X — ISP-provided CPE (FTTH PPPoE on VLAN 835, VoIP on FXS)
- Cat6A drops, conduit-routed
- TP-Link EAP225 V3 — multi-SSID/VLAN Wi-Fi access point
**Software stack**
- Proxmox VE 8 — type-1 hypervisor
- OPNsense 26.x — pfSense-based firewall, FreeBSD core
- Ubuntu Server 24.04 LTS — application VM
- Node.js 22 LTS, Nginx 1.24, PM2, SQLite (better-sqlite3)
- Let's Encrypt with Certbot
- Cloudflare Registrar + DNS, ddclient via OPNsense plugin
- Rclone + GnuPG (AES256) — backup encryption and cloud upload
---

## The journey
What follows is roughly chronological.

### Phase 1 — Audit and design
 
Mapped what was in the office network: 4 PCs, 1 printer, 4 IP cameras, 2 VoIP handsets, 1 robot mower, 2 irrigation actuators, 1 server hosting a vehicle-management webapp running on macOS. Drew the existing topology on paper — a single broadcast domain — and the target topology with five VLANs, motivating each separation:
 
- **Mgmt** isolated so a compromised office PC can't reach the firewall GUI or the hypervisor
- **Primary** for office workstations, where users authenticate and print
- **Secondary** for IoT, because consumer-grade cameras and a mower's firmware are not trustworthy hosts
- **Guest** for visitors, with no path to anything internal
- **DMZ** for the soon-to-be-public webapp, which by definition will receive inbound traffic from anywhere

### Phase 2 — Physical layer
 
Pulled Cat6A drops where needed, terminated the ends, replaced the unmanaged switch with a Netgear GS308E to get 802.1Q. Configured the switch:
 
- Port 6 — trunk to OPNsense LAN (VID 1 untagged + 10/20/30/40 tagged)
- Ports 1–5 — access ports for office VLAN 10
- Port 7 — access VLAN 1 (admin laptop)
- Port 8 — access VLAN 20 (IoT & cameras)

### Phase 3 — OPNsense and segmentation
 
Built OPNsense as a Proxmox VM (4 GB RAM, 16 GB disk, 2 NICs). Bridge `vmbr0` set to `bridge-vlan-aware yes` with `bridge-vids 2-4094` to let 802.1Q tagging pass through to the VM.
 
For each VLAN: a logical interface, a /24, Kea DHCP, anti-lockout rule on management, then the firewall rules. The DMZ rule that does the heavy lifting:
 
```
block in DMZ → 192.168.0.0/16   log
pass  in DMZ → any              # everything else (= internet)
```
 
Then a more specific block above it: `block DMZ → This Firewall` so the webapp VM can never speak to the OPNsense GUI on any of its IPs.
 
### Phase 4 — Application hosting
 
Provisioned an Ubuntu 24.04 VM on Proxmox tagged into VLAN 40:
 
```
qm create 200 \
  --name webapp --memory 6144 --cores 4 --cpu host \
  --net0 virtio,bridge=vmbr0,tag=40 \
  --scsihw virtio-scsi-single --scsi0 local-lvm:40 \
  --ostype l26 --agent enabled=1 --onboot 1
```
 
Static IP 192.168.140.10/24, gateway 192.168.140.1 (the OPNsense DMZ interface).

### Phase 5 — Application deployment
 
Deployed a Node.js / Express + SQLite webapp behind nginx as a reverse proxy:
 
```
[browser]  →  nginx :443  →  Node :3001  →  SQLite (data/*.db)
                                             uploads/*.{pdf,jpg}
```
 
PM2 manages the Node process with auto-restart and `pm2 startup` integrating into systemd so it survives reboots. The app's existing data was migrated from the old macOS host with a careful step that's easy to miss:
 
```bash
sqlite3 vehicle_management.db "PRAGMA wal_checkpoint(TRUNCATE);"
```
 
SQLite in WAL mode keeps recent writes in `*-wal` and `*-shm` sidecar files. Copying just `*.db` would have lost six weeks of changes. The checkpoint forces a consistent merge into the main DB before transferring it via SCP.

 ### Phase 6 — Host hardening
 
Defense-in-depth on the application VM, top to bottom:
 
- **SSH** — generated an Ed25519 key on my laptop, deployed via `ssh-copy-id`, **then** disabled password auth in `/etc/ssh/sshd_config.d/50-cloud-init.conf` (which on Ubuntu 24.04 overrides the main `sshd_config` — this is the gotcha most guides miss).
- **UFW** — default-deny inbound, allow outbound, explicit allow on 22 / 80 / 443. Port 3001 (Node) deliberately not exposed; nginx talks to it on 127.0.0.1 only.
- **Fail2ban** — `[sshd]` jail with `backend=systemd` (correct for Ubuntu 24.04, which logs SSH via journald), 3 strikes / 10-minute window, exponential ban escalation up to 7 days. Whitelist for management VLAN to avoid self-locking-out.
- **Nginx** — security headers: `Strict-Transport-Security`, `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`, `Content-Security-Policy`. Hidden `server_tokens`. `client_max_body_size 50M` for file uploads.

### Phase 7 — Backup strategy
 
A backup that's never been restored is not a backup. Wrote a bash script (`backup.sh`) that runs nightly at 03:00 via cron and does:
 
1. **Online SQLite backup** — `sqlite3 source.db ".backup dest.db"` for a consistent snapshot without stopping the app
2. **Tar + gzip** the uploads directory
3. **Bundle** with a metadata file (timestamp, host, git commit hash of the deployed app)
4. **GPG symmetric encryption** with AES256, passphrase from a 0600 file outside the repo
5. **Upload to Google Drive** via rclone, OAuth-authenticated, automated.
6. **Tiered rotation**: 7 daily, 4 weekly (Sunday), 6 monthly (1st of month). Old backups deleted automatically.
7. **Telegram notification** — success message with size and tier, or failure with log path
Then I tested a restore. Downloaded the latest backup, decrypted with the passphrase, extracted, ran `PRAGMA integrity_check;` on the DB, counted rows, sanity-checked the uploads count. All green.

### Phase 8 — Public exposure
 
Bought `<domain>.app` on Cloudflare Registrar (the `.app` TLD is HSTS-preloaded by all major browsers, so HTTPS is non-negotiable from day one — even before the cert exists, browsers refuse `http://`).
 
The path from internet to webapp:
```
client  →  <domain>.app  →  x.x.x.x (TIM public)
        →  TIM HUB+ DMZ Host
        →  192.168.1.78 (OPNsense WAN)
        →  Destination NAT 80/443
        →  192.168.140.10 (webapp VM)
        →  nginx → Node
```
 
**Dynamic IP problem.** The TIM line uses PPPoE with a dynamic public IP. Solved with the `os-ddclient` plugin on OPNsense, configured to talk to the Cloudflare API v4 with a scoped API token (`Edit zone DNS` only on `<domain>.app`, no other zones). Check IP method: `cloudflare-ipv4`, which avoids the trap of reading the WAN interface IP — useless when the firewall sits behind another NAT.
 
**Cert.** Let's Encrypt via certbot.
 
**Final HTTPS config in nginx**: HSTS `max-age=31536000; includeSubDomains`, modern cipher suite (defaults from `/etc/letsencrypt/options-ssl-nginx.conf` are fine), `http2` enabled.


## Incident Stories
These are the moments where the project stopped being a guide-follow and started being engineering. Each one taught me something.

### 1. The silent firewall drop
 
**Symptom.** From a VLAN 20 client, ping to its own gateway worked. Ping to anything else timed out. Firewall rules looked correct in the GUI.
 
**Investigation.** Three layers of `tcpdump` simultaneously:
 
```bash
# On OPNsense
tcpdump -i vtnet1 icmp           # at the VM's NIC
# On Proxmox host
tcpdump -i vmbr0 -e vlan         # at the bridge
tcpdump -i tap100i0              # on the OPNsense VM's tap
```
 
Packets reached OPNsense intact, with the right VLAN tag. They never made it back. So pf was dropping silently.
 
```
pfctl -s rules | grep block
```
 
revealed three "block" rules whose **destination was unset** (= any). Because they used `quick`, they matched and short-circuited before the legitimate `pass` rule below. The GUI had hidden the problem because the destination field was blank-and-therefore-implicitly-any, which on the rules listing looks the same as "destination: any" intentional.
 
**Fix.** Set explicit destinations (`Net_Primary`, `Net_Guest`, `Net_DMZ` aliases). Enabled `Log` on all three.
 
**Lesson.** Always specify destination on block rules. Always log them. The GUI is not a substitute for `pfctl -s rules`.

### 2. Direct-WAN cutover failure
 
**Plan.** Replace the TIM HUB+ entirely. Configure OPNsense to do its own PPPoE on VLAN 835, MAC-clone the HUB+, become the only CPE.
 
**What happened.** Verified `tcpdump` showed correctly tagged 835 frames going out of the OPNsense WAN NIC. TIM's BRAS never responded to DHCP. Even with the HUB+ powered off, the lease wouldn't release. TIM uses TR-069 to provision the CPE and locks the line to the authorized device — MAC-cloning isn't enough.
 
**Pivot.** Accepted double-NAT: TIM HUB+ stays as the WAN router (and importantly, keeps doing VoIP via FXS, which is configured by TIM through TR-069 and doesn't survive replacement); OPNsense becomes an internal firewall on a 192.168.1.x WAN side. For the webapp's public-facing role, set the TIM HUB+ to "DMZ Host" pointing to OPNsense, and OPNsense forwards 80/443 onward.
 
**Lesson.** "I'll just replace the ISP CPE" is rarely as simple as it looks on Italian FTTH. Pragmatic > pure.

### 3. The DHCP port 67 conflict
 
**Symptom.** After an OPNsense reboot, no client got an IP.
 
**Investigation.**
 
```
sockstat -4 -l | grep :67
```
 
showed `dnsmasq` listening on port 67, not Kea. Both services were enabled. Dnsmasq grabbed the socket first.
 
**Fix.** Disabled the dnsmasq service entirely (Services → Dnsmasq → uncheck Enable). Kept Unbound as the resolver and Kea as the DHCP server.
 
**Lesson.** Two DHCP servers on the same broadcast domain is a classic CTF / pentest finding. On a single host, "service started but won't bind" is often a port collision; `sockstat` (FreeBSD) or `ss -tlnp` (Linux) tells you who has it.

## References
- Seth Enoka, *Cybersecurity for Small Networks* — No Starch Press, 2022. Foundational book for this project.
- ISA-99 / IEC 62443 — reference architectures for industrial network segmentation. Not directly applicable to a tiny office, but the zone-and-conduit thinking translates down.
- OWASP Application Security Verification Standard — checklist used to audit the webapp configuration.
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) — used for the nginx TLS settings.
---

## Contact
Always happy to discuss the decisions in this project.

