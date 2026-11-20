# Cybersecurity Home Lab

A fully virtualized home lab (VirtualBox), built to develop hands-on skills in network administration, penetration testing (Red Team), and detection (Blue Team) — as part of a portfolio and internship search.

## ⚠️ Legal and ethical notice

All attacks, scans, and tests described in this repository are performed **exclusively** on personal virtual machines, isolated from the internet, in a closed lab environment. None of these techniques are applied to third-party systems without authorization. This repository is for educational purposes only.

## Goals

- Build a segmented network infrastructure with a firewall (pfSense)
- Practice reconnaissance and vulnerability analysis (Red Team)
- Set up detection and monitoring (Blue Team)
- Document every step like a real professional report

## Stack

- **Virtualization**: VirtualBox
- **Firewall / router**: pfSense CE 2.6.0
- **Attacker**: Kali Linux (Dual-homed: eth0 = NAT/WAN, eth1 = net-attackers/Lab)
- **Targets**: Metasploitable2, Ubuntu Server (Apache), Windows Server (upcoming)
- **Offensive tooling**: Nmap, Metasploit Framework, Searchsploit, VNCViewer
- **Detection**: Wazuh (upcoming)

## Architecture

See [`docs/architecture.md`](docs/architecture.md) for the detailed network diagram and IP addressing plan.

## Repository structure

```
homelab-cyber/
├── docs/
│   ├── architecture.md      # Network diagram, zones, IP addressing plan
│   ├── journal.md           # Step-by-step lab notebook
│   └── firewall-rules.md    # pfSense rules and their justification
├── screenshots/
│   ├── network/              # Network configuration captures
│   ├── recon/                 # Scan captures (Nmap, etc.)
│   └── rules/                  # Firewall rule captures
└── reports/                   # Final reports (pentest, etc.)
```

## Progress

- [x] Segmented network (4 zones: WAN, Servers, Users, Attackers)
- [x] pfSense installed and configured
- [x] First firewall rules (targeted ICMP & HTTP to Ubuntu)
- [x] Metasploitable reconnaissance (Nmap service & version scan)
- [x] Vulnerability research and credential enumeration (Searchsploit, Metasploit)
- [x] Metasploitable compromise & root access across multiple vectors:
  - [x] PostgreSQL UDF injection to Meterpreter (`postgres` user) + SUID Nmap LPE to `root`
  - [x] Samba `usermap_script` (CVE-2007-2447) direct remote `root` shell
  - [x] VNC default credential compromise (`password`) & X11 root desktop
- [x] Attacker multi-homing & network stabilization (Kali dual-adapter NAT + Lab routing)
- [ ] Active Directory (Windows Server)
- [ ] SIEM / detection (Wazuh)
- [ ] Hardening and re-testing
- [ ] Final penetration test report

## Author

Mohamed Chemseddin BOUFARIS - Cybersecurity student @ENSIAS
