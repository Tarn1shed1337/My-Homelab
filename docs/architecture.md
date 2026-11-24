# Network architecture

## Diagram

pfSense acts as the central router/firewall between 4 zones:

- **WAN**: internet access via VirtualBox NAT
- **SERVERS** (LAN): target servers
- **USERS** (OPT1): user workstations (Windows 10 Pro client)
- **ATTACKERS** (OPT2): attack machine (Kali)

## IP addressing plan

| Zone | VirtualBox internal network | Subnet | Gateway (pfSense) | pfSense interface |
|---|---|---|---|---|
| Internet | NAT | 10.0.2.0/24 | 10.0.2.15 (DHCP) | WAN (em0) |
| Servers | net-serveurs | 10.0.10.0/24 | 10.0.10.1 | LAN (em1) |
| Users | net-users | 10.0.20.0/24 | 10.0.20.1 | OPT1 / USERS (em2) |
| Attackers | net-attackers | 10.0.30.0/24 | 10.0.30.1 | OPT2 / ATTACKERS (em3) |

> Renamed from `net-attaquants` to `net-attackers` on 2026-11-19 (updated identically on both the VirtualBox adapter and the pfSense-side name).

## Machines

| Machine | Zone | IP | Role |
|---|---|---|---|
| pfSense | all | .1 in each zone | Firewall / router |
| Kali | Attackers | 10.0.30.10/24 | Attack machine; dual-homed (eth0 = NAT for internet, eth1 = lab network, with `ipv4.never-default` on eth1 and a static route to 10.0.10.0/24 via 10.0.30.1) |
| Ubuntu | Servers | 10.0.10.20/24 | Test server (Apache) |
| Metasploitable2 | Servers | 10.0.10.30/24 | Intentionally vulnerable target |
| Windows Server 2019 (DC01) | Servers | 10.0.10.40/24 | Domain Controller, domain `label.local`. Roles: AD DS, DNS. OU `LAB` holds all test accounts. |
| Windows 10 Pro (client) | Users | 10.0.20.10/24 | Domain-joined to `label.local`. Gateway: 10.0.20.1, DNS: 10.0.10.40 (DC). Simulates a standard user workstation. |

## Active Directory Deliberately Vulnerable Objects (OU `LAB`)

| Account | Misconfiguration | Purpose |
|---|---|---|
| `svc-sql` | SPN registered (`MSSQLSvc/dc01.label.local:1443`) | Kerberoasting target |
| `asrep-test` | Kerberos pre-authentication disabled (`DONT_REQ_PREAUTH`) | AS-REP Roasting target |
| `acl-test` | `GenericAll` on `Domain Admins` group object | ACL abuse / privilege escalation target |

## Active Directory Firewall Rules (pfSense)

| Rule | Interface | Source | Destination | Port | Purpose |
|---|---|---|---|---|---|
| Pass TCP/UDP 88 | OPT2 (ATTACKERS, em3) | ATTACKERS net | 10.0.10.40 | 88 (Kerberos) | Required for AS-REP Roasting / Kerberoasting from Kali |
| Pass TCP/UDP 389 | OPT2 (ATTACKERS, em3) | ATTACKERS net | 10.0.10.40 | 389 (LDAP) | Required for ACL abuse / directory writes (bloodyAD) from Kali |
| Pass any | OPT1 (USERS, em2) | USERS net | 10.0.10.40 | any | Legitimate domain join & AD functionality for Windows client |
| Pass TCP/UDP 53 | OPT2 (ATTACKERS, em3) | ATTACKERS net | 10.0.10.40 | 53 (DNS) | Required for `bloodhound-python` collector to resolve `dc01.label.local` |

## Technical Implementation Notes

- **VirtualBox Host-Time Sync**: Disabled for the DC01 VM (`VBoxManage setextradata "<VM>" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 1`) after it repeatedly overwrote manual clock corrections needed for Kerberos authentication to succeed from Kali.
- **Active Directory Web Services (ADWS)**: Required a manual service restart (`Restart-Service ADWS`) after initial promotion — PowerShell `Get-ADUser` attribute queries returned blank values until the cache was refreshed.
- **BloodHound CE & Remote Collector**: BloodHound Community Edition is installed on Kali (`bloodhound-start`), backed by PostgreSQL 18 + Neo4j, with the web UI accessible on `http://127.0.0.1:8080`. Data collection runs remotely via `bloodhound-python` in `DCOnly` mode (pure LDAP queries against DC01) — no agent or footprint is required on the domain-joined client or DC.

## Design principles

- Each zone is isolated by default: pfSense blocks all inbound traffic on OPT interfaces until a rule explicitly allows it.
- Metasploitable always runs on an internal network (never bridged), to avoid any accidental exposure.
- DHCP is disabled on every interface: every machine has a documented static IP.
