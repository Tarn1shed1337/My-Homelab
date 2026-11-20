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

## Day 2 — 2026-11-19

### Goals
- Triage Metasploitable's exposed services and conduct vulnerability & credential research.
- Exploit identified vulnerabilities across multiple vectors (database, file sharing, remote access) to achieve root.
- Fix a networking issue on Kali (lost internet access) and establish stable dual-homed routing.

### What I did
- Picked 3 services from the Day 1 Nmap scan to investigate: MySQL 5.0.51a (3306), PostgreSQL 8.3.0-8.3.7 (5432), and ProFTPD 1.3.1 (2121).
- Used `searchsploit` on each service, cross-checking exact affected version strings against advisory texts rather than trusting titles alone. Confirmed a DoS vulnerability on MySQL (< 5.1.49) and noted ProFTPD 1.3.1 had no direct unauthenticated remote exploits.
- Identified PostgreSQL's User-Defined Function (UDF) command execution (`linux/local/7855.txt`) as a prime exploit candidate, noting it requires authenticated database access first.
- Attempted manual connection using `psql -h 10.0.10.30 -U msfadmin`; rejected due to SSL protocol mismatch (modern Kali client vs. legacy SSL on Metasploitable2).
- Used Metasploit's `auxiliary/scanner/postgres/postgres_login` (`CreateSession true`) to brute-force default credentials: authenticated with `postgres:postgres` and opened an interactive SQL session (Session 1).
- Learned the session's command syntax (`query <SQL>` instead of raw SQL), successfully enumerating databases (`SELECT datname FROM pg_database;` $\rightarrow$ `template1`, `template0`, `postgres`).
- Ran `auxiliary/scanner/postgres/postgres_hashdump` on the session, successfully extracting the password hash (`postgres:md53175bce1d3201d16594cebf9d7eb3f9d`).
- Leveraged `exploit/linux/postgres/postgres_payload`: Metasploit uploaded a shared library (`/tmp/BZmzRFSa.so`) via UDF injection, spawning a reverse Meterpreter session (Session 3) running as service user `postgres` (`uid=108`).
- Dropped into a system shell from Meterpreter, discovered an SUID binary `/usr/bin/nmap` (v4.53), and broke out to an interactive root shell using `/usr/bin/nmap --interactive` followed by `!sh` (`euid=0(root)`, `whoami` $\rightarrow$ `root`). Dropped a proof-of-compromise flag in `/root/Desktop/pwned`.
- Researched Samba 3.0.20 via `searchsploit` and Metasploit (`exploit/multi/samba/usermap_script`, CVE-2007-2447): triggered remote command execution in Samba's username mapping script, spawning an immediate unauthenticated reverse shell (Session 4) running directly as `root` (`uid=0(root) gid=0(root)`).
- Tested VNC (port 5900) with `auxiliary/scanner/vnc/vnc_login`: uncovered default password `password`. Connected via `vncviewer 10.0.10.30` from Kali, obtaining full graphical X11 desktop access (TightVNC `metasploitable:0`) with an active root terminal.
- Lost internet access on Kali; diagnosed it as `eth0` (NAT) being disconnected while `eth1` (lab network) still worked. Renamed the attackers' internal network from `net-attaquants` to `net-attackers` (updated identically in VirtualBox and pfSense).
- Configured Linux longest-prefix-match routing: set `ipv4.never-default` on `eth1` to prevent it from competing with `eth0` for the default gateway, while binding the static route to the servers subnet (`10.0.10.0/24 via 10.0.30.1 dev eth1`).
- Hit transient duplicate/corrupted ping replies after toggling the virtual cable — resolved by properly reconnecting the adapter and letting ARP settle; confirmed full connectivity across both WAN and the lab.

### Problems & fixes
- `searchsploit <product>` without a version returns every version's exploits — must filter manually or re-run with the exact version string.
- `psql` client failed with `SSL error: unsupported protocol` — caused by modern OpenSSL rejecting legacy SSLv2/v3. Metasploit's auxiliary module handled legacy SSL negotiation cleanly.
- `SHOW DATABASES;` caused syntax error in PostgreSQL session — recognized MySQL vs. PostgreSQL syntax difference and used PostgreSQL catalog query `SELECT datname FROM pg_database;`.
- Kali's `eth0` showed as `disconnected` because the virtual cable was unplugged in VirtualBox from an earlier test; reconnecting it and configuring `ipv4.never-default` on `eth1` restored stable multi-homing.
- Flapping the virtual cable left stale ARP entries, producing corrupted/duplicate ping replies — resolved once the connection stabilized.

### What I learned
- An exploit title's version range (`< X`) only means the bug was *introduced* at some point before X and fixed in X — it doesn't mean every older version is affected. Always check the advisory text.
- Credential enumeration (testing default/weak logins) is a standard, necessary step before any further access — not something to skip by just looking up documented defaults.
- A single target presents multiple independent compromise chains: service authentication + UDF code execution + SUID binary local privilege escalation (`postgres` $\rightarrow$ `/usr/bin/nmap --interactive` $\rightarrow$ `root`), direct unauthenticated daemon RCE (`samba/usermap_script`), and administrative service default credentials (VNC `password`).
- A multi-homed Linux box (two NICs) uses the most specific matching route for each destination; the default route only handles traffic with no more specific match. Setting `ipv4.never-default` on internal lab adapters prevents routing confusion with WAN NAT.

### Next
- Begin deployment of Windows Server in the Users zone for Active Directory exercises.
- Set up Blue Team detection and monitoring infrastructure (Wazuh SIEM) to log and detect these attack patterns.
- Consolidate findings (service, version, CVE, credentials found, exploit chain) into a professional pentest report.

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
