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
- **Firewall / router**: pfSense CE
- **Attacker**: Kali Linux
- **Targets**: Metasploitable2, Ubuntu Server, Windows Server (upcoming)
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
├── screenshots/              # Screenshots
└── reports/
    └── pentest-report.md    # Comprehensive penetration test report (Metasploitable2)
```

## Progress

- [x] Segmented network (4 zones: WAN, Servers, Users, Attackers)
- [x] pfSense installed and configured
- [x] First firewall rule (targeted ICMP)
- [x] Metasploitable reconnaissance (Nmap)
- [x] Vulnerability research, credential enumeration, and exploitation on exposed services
- [x] Web application security assessment (DVWA — Directory indexing, Brute-forcing, OS Command Injection)
- [x] Final penetration test report ([`reports/pentest-report.md`](reports/pentest-report.md))
- [ ] Active Directory (Windows Server)
- [ ] SIEM / detection (Wazuh)
- [ ] Hardening and re-testing

## Author

Mohamed Chemseddin BOUFARIS — Cybersecurity student
