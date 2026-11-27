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
| Kali | Attackers | 10.0.30.10/24 | Attack machine; dual-homed (eth0 = NAT for internet, eth1 = lab network, with `ipv4.never-default` on eth1 and static routes to `10.0.10.0/24 via 10.0.30.1` [SERVERS] and `10.0.20.0/24 via 10.0.30.1` [USERS, added 2026-11-25]) |
| Ubuntu | Servers | 10.0.10.20/24 | Test server (Apache) |
| Metasploitable2 | Servers | 10.0.10.30/24 | Intentionally vulnerable target |
| Windows Server 2019 (DC01) | Servers | 10.0.10.40/24 | Domain Controller, domain `label.local`. Roles: AD DS, DNS. OU `LAB` holds all test accounts. |
| Windows 10 Pro (client) | Users | 10.0.20.10/24 | Domain-joined to `label.local` (`DESKTOP-C8VMV6Q`, OS Build `19045.2965` / 22H2). Gateway: 10.0.20.1, DNS: 10.0.10.40 (DC). Monitored by Wazuh agent `Windows-agent` (ID `001`, v4.14.11). |
| Ubuntu Server (Wazuh SIEM) | Servers | 10.0.10.50/24 | Dedicated Blue Team SIEM server running Wazuh v4.14.11 all-in-one (Indexer, Manager, Dashboard, Filebeat); HTTPS web UI on TCP 443. |

## Active Directory Deliberately Vulnerable Objects (OU `LAB`)

| Account | Misconfiguration | Purpose |
|---|---|---|
| `svc-sql` | SPN registered (`MSSQLSvc/dc01.label.local:1443`) | Kerberoasting target |
| `asrep-test` | Kerberos pre-authentication disabled (`DONT_REQ_PREAUTH`) | AS-REP Roasting target |
| `acl-test` | `GenericAll` on `Domain Admins` group object; initially shared identical password/NTLM hash (`64f12cdd...`) with `compte2` *(remediated on 2026-11-27: password rotated + added to `Protected Users` group)* | ACL abuse → Domain Admin + lateral movement / hardening re-test target |
| `compte1` | Standard domain user | Joined the Windows 10 client to `label.local` (default `ms-DS-MachineAccountQuota`, no admin rights needed) |
| `compte2` | Standard domain user; added to Windows 10 local `Administrators` on 2026-11-25 | Simulates an attacker with local admin access on the user workstation |

## Active Directory & SIEM Firewall Rules (pfSense)

| Rule | Interface | Source | Destination | Port | Purpose |
|---|---|---|---|---|---|
| Pass TCP/UDP 88 | OPT2 (ATTACKERS, em3) | ATTACKERS net | 10.0.10.40 | 88 (Kerberos) | Required for AS-REP Roasting / Kerberoasting from Kali |
| Pass TCP/UDP 389 | OPT2 (ATTACKERS, em3) | ATTACKERS net | 10.0.10.40 | 389 (LDAP) | Required for ACL abuse / directory writes (`bloodyAD`) from Kali |
| Pass any | OPT1 (USERS, em2) | USERS net | 10.0.10.40 | any | Legitimate domain join & AD functionality for Windows client |
| Pass TCP/UDP 53 | OPT2 (ATTACKERS, em3) | ATTACKERS net | 10.0.10.40 | 53 (DNS) | Required for `bloodhound-python` collector to resolve `dc01.label.local` |
| Pass TCP 135, 445 | OPT2 (ATTACKERS, em3) | ATTACKERS net | 10.0.10.40, 10.0.20.10 | 135 (MSRPC), 445 (SMB) | Required for BloodHound `-c All`/`-c Session` enumeration and `impacket-psexec` Pass-the-Hash |
| Pass TCP 1514, 1515 | OPT1 (USERS, em2) | USERS net | 10.0.10.50 | 1514, 1515 | Wazuh agent event telemetry (`1514`) and enrollment (`1515`) from Windows 10 client to Wazuh Manager |

## Technical Implementation Notes

- **VirtualBox Host-Time Sync**: Disabled for the DC01 VM (`VBoxManage setextradata "<VM>" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 1`) after it repeatedly overwrote manual clock corrections needed for Kerberos authentication to succeed from Kali.
- **Active Directory Web Services (ADWS)**: Required a manual service restart (`Restart-Service ADWS`) after initial promotion — PowerShell `Get-ADUser` attribute queries returned blank values until the cache was refreshed.
- **BloodHound CE & Collection Modes**: BloodHound Community Edition is installed on Kali (`bloodhound-start`), backed by PostgreSQL 18 + Neo4j, with the web UI accessible on `http://127.0.0.1:8080` (Neo4j password `neo4j1`). `-c DCOnly` queries LDAP against DC01 only; `-c All` / `-c Session` additionally enumerates hosts live over SMB (445) / RPC (135) for active sessions (`HasSession`) and local admins. Note: remote `NetSessionEnum` is restricted by modern Windows OS hardening to elevated accounts (`Access denied ... likely a patched OS`), which is expected real-world behavior rather than a lab misconfiguration.
- **Static Route Troubleshooting**: If `nmap` reports `filtered` despite verified pfSense rules and disabled Windows Firewall profiles, inspect `ip route` and `tcpdump` on Kali (`0 packets captured` indicates a missing static route for the target subnet rather than firewall drops).
- **Windows Firewall Per-Profile Scope (`RemoteAddress`)**: Built-in Windows Firewall rules such as `File and Printer Sharing (SMB-In)` apply different remote address scopes per profile (`Domain` = `Any`, whereas `Private`/`Public` = `LocalSubnet` by default). Cross-subnet testing when a host falls back to the `Private` profile requires explicitly expanding `RemoteAddress` via `Set-NetFirewallRule`.
- **Wazuh v4.14 Configuration & Active Response Layering**: In Wazuh 4.14, the vulnerability module XML block is named `<vulnerability-detection>` (formerly `<vulnerability-detector>`) and relies on `syscollector` inventory paired with `nvd` and `msu` feeds. For Windows Active Response against inbound network attacks, host firewall blocking (`netsh.exe`) proved reliable (`open` → `filtered`), whereas routing-table null-routing (`route-null.exe`) executed cleanly in logs without preventing inbound TCP handshakes.

## Design principles

- Each zone is isolated by default: pfSense blocks all inbound traffic on OPT interfaces until a rule explicitly allows it.
- Metasploitable always runs on an internal network (never bridged), to avoid any accidental exposure.
- DHCP is disabled on every interface: every machine has a documented static IP.
