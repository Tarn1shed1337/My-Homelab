# Network architecture

## Diagram

pfSense acts as the central router/firewall between 4 zones:

- **WAN**: internet access via VirtualBox NAT
- **SERVERS** (LAN): target servers
- **USERS** (OPT1): user workstations (upcoming)
- **ATTACKERS** (OPT2): attack machine (Kali)

## IP addressing plan

| Zone | VirtualBox internal network | Subnet | Gateway (pfSense) | pfSense interface |
|---|---|---|---|---|
| Internet | NAT | 10.0.2.0/24 | 10.0.2.15 (DHCP) | WAN (em0) |
| Servers | net-serveurs | 10.0.10.0/24 | 10.0.10.1 | LAN (em1) |
| Users | net-users | 10.0.20.0/24 | 10.0.20.1 | OPT1 / USERS (em2) |
| Attackers | net-attaquants | 10.0.30.0/24 | 10.0.30.1 | OPT2 / ATTACKERS (em3) |

## Machines

| Machine | Zone | IP | Role |
|---|---|---|---|
| pfSense | all | .1 in each zone | Firewall / router |
| Kali | Attackers | 10.0.30.10/24 | Attack machine |
| Ubuntu | Servers | 10.0.10.20/24 | Test server (Apache) |
| Metasploitable2 | Servers | 10.0.10.30/24 | Intentionally vulnerable target |
| Windows Server (AD) | TBD | TBD | Upcoming |

## Design principles

- Each zone is isolated by default: pfSense blocks all inbound traffic on OPT interfaces until a rule explicitly allows it.
- Metasploitable always runs on an internal network (never bridged), to avoid any accidental exposure.
- DHCP is disabled on every interface: every machine has a documented static IP.
