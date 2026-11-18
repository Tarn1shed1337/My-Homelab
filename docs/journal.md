# Lab notebook — day by day

Each day gets one entry. Copy the template at the bottom when starting a new day.

---

## Day 1 — 2026-11-18

### Goals
- Build the base segmented network and get pfSense routing between zones.

### What I did
- Created 3 VirtualBox internal networks (net-serveurs, net-users, net-attaquants) + NAT for the WAN.
- Installed pfSense CE 2.6.0, assigned the 4 interfaces (WAN, LAN, OPT1, OPT2).
- Assigned IPs: LAN 10.0.10.1, OPT1 10.0.20.1, OPT2 10.0.30.1.
- Configured static IPs on Kali (10.0.30.10) and Ubuntu (10.0.10.20), with a route from Kali to the servers zone via pfSense.
- Accessed the pfSense WebGUI from Ubuntu and completed the setup wizard.
- Created a Pass ICMP rule on ATTACKERS (source ATTACKERS net → destination 10.0.10.20) and verified it: ping to 10.0.10.20 succeeds (TTL=63, confirming it passed through pfSense), ping to 10.0.10.1 stays blocked.
- Added a Pass TCP port 80 rule and installed Apache on Ubuntu: watched Nmap's report on 10.0.10.20 move from `filtered` (no rule) to `closed` (rule but no service) to `open` (service running).
- Set a static IP on Metasploitable (10.0.10.30), took a clean snapshot, added a broad ATTACKERS → 10.0.10.30 rule, and ran `nmap -sV 10.0.10.30`: 23 open ports out of 1000 scanned, with service versions identified (vsftpd 2.3.4, OpenSSH 4.7p1, Samba, MySQL, PostgreSQL, etc.).

### Problems & fixes
- First IP assignment on pfSense's OPT interfaces was swapped (ATTACKERS and USERS zones reversed) — found by comparing VirtualBox adapter order with pfSense's MAC addresses.
- Ubuntu's `netplan-enp0s3` connection silently ignored `nmcli modify ... manual` — fixed by creating a fresh nmcli connection instead of editing the netplan-generated one.

### What I learned
- A firewall rule only controls whether a packet reaches a host — whether a port is actually open still depends on a service listening there.
- `filtered` / `closed` / `open` each mean something different: no response, response but no service, response with a service.
- Promiscuous mode only matters for sniffing traffic on a virtual network segment; it has no effect on basic routing or static IP configuration.

### Next
- Triage Metasploitable's exposed services and start documenting them for the report.

---

## Template for new entries

```
## Day N — YYYY-MM-DD

### Goals
- 

### What I did
- 

### Problems & fixes
- 

### What I learned
- 

### Next
- 
```
