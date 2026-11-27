# pfSense firewall rules

## Default behavior

- Any OPT interface (ATTACKERS, USERS) blocks all inbound traffic by default.
- The LAN (SERVERS) allows all outbound traffic by default (standard pfSense rule).

## Active rules

| Interface | Action | Protocol | Source | Destination | Port | Justification |
|---|---|---|---|---|---|---|
| ATTACKERS | Pass | ICMP | ATTACKERS net | 10.0.10.20 | — | Targeted connectivity test (ping) to Ubuntu |
| ATTACKERS | Pass | TCP | ATTACKERS net | 10.0.10.20 | 80 | HTTP access test to Ubuntu |
| ATTACKERS | Pass | * | ATTACKERS net | 10.0.10.30 | * | Broad access to Metasploitable (intentionally exposed lab target) |
| ATTACKERS | Pass | TCP/UDP | ATTACKERS net | 10.0.10.40 | 88 | Kerberos authentication (AS-REP Roasting & Kerberoasting from Kali) |
| ATTACKERS | Pass | TCP/UDP | ATTACKERS net | 10.0.10.40 | 389 | LDAP directory access & attribute modification (`bloodyAD` ACL abuse) |
| ATTACKERS | Pass | TCP/UDP | ATTACKERS net | 10.0.10.40 | 53 | DNS resolution for BloodHound collector (`bloodhound-python`) |
| ATTACKERS | Pass | TCP | ATTACKERS net | 10.0.10.40, 10.0.20.10 | 135, 445 | MSRPC & SMB access for BloodHound `-c All` enumeration and `impacket-psexec` Pass-the-Hash |
| USERS | Pass | * | USERS net | 10.0.10.40 | * | Full access to DC for legitimate domain join & client communication |
| USERS | Pass | TCP | USERS net | 10.0.10.50 | 1514, 1515 | Wazuh agent event collection (`1514`) and agent enrollment (`1515`) to Wazuh SIEM |

## Temporary Validation & Maintenance Exceptions

| Interface | Action | Protocol | Source | Destination | Port | Scope / Lifecycle |
|---|---|---|---|---|---|---|
| ATTACKERS | Pass | TCP | ATTACKERS net | 10.0.20.10 | 445 | Temporary exception used during Blue Team testing to replay cross-zone SMB brute-force (`nxc`) and validate Wazuh Active Response (`netsh`) |
| USERS | Pass | TCP | USERS net | * (Internet) | 443 | Temporary maintenance exception opened during the Hardening phase to verify Windows Update status on the isolated Windows 10 client |

## Observations

- A `filtered` port (Nmap) means the firewall is silently dropping traffic: no information about the real state of the service.
- A `closed` port means the packet passes the firewall but no service is listening.
- An `open` port means the packet passes and a service responds.
