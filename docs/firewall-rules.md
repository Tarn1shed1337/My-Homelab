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

## Observations

- A `filtered` port (Nmap) means the firewall is silently dropping traffic: no information about the real state of the service.
- A `closed` port means the packet passes the firewall but no service is listening.
- An `open` port means the packet passes and a service responds.
