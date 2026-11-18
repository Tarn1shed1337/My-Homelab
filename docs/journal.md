# Lab notebook

## 2026-11-18 — Base network

- Created 3 VirtualBox internal networks (net-serveurs, net-users, net-attaquants) + NAT for the WAN.
- Installed pfSense CE 2.6.0, assigned the 4 interfaces (WAN, LAN, OPT1, OPT2).
- Assigned IPs: LAN 10.0.10.1, OPT1 10.0.20.1, OPT2 10.0.30.1.
- Configured a static IP on Kali (10.0.30.10) with a route to the servers zone via pfSense.
- Configured a static IP on Ubuntu (10.0.10.20) via NetworkManager.
- Accessed the pfSense WebGUI from Ubuntu, completed the setup wizard.

## 2026-11-18 — First firewall rule and tests

- Created a Pass ICMP rule on the ATTACKERS interface, source ATTACKERS net, destination 10.0.10.20.
- Test: ping to 10.0.10.20 succeeds (TTL=63, confirming it passed through pfSense), ping to 10.0.10.1 blocked (outside the rule).
- Nmap scan (`nmap -p 80 10.0.10.20`): port shown as `filtered` (firewall silently drops the packet).
- Added a Pass TCP port 80 rule: the port changes to `closed` (no service), then `open` after installing Apache.

## 2026-11-18 — Metasploitable reconnaissance

- Configured a static IP on Metasploitable (10.0.10.30) via `/etc/network/interfaces`.
- Took a clean-state snapshot before any attack.
- Added a broad pfSense rule (ATTACKERS → 10.0.10.30, all traffic allowed): target intentionally exposed for the exercise.
- `nmap -sV 10.0.10.30` scan: 23 open ports out of 1000 scanned (977 closed), services identified with versions (vsftpd 2.3.4, OpenSSH 4.7p1, Samba, MySQL, PostgreSQL, etc.).
