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
- Triage Metasploitable's exposed services and start credential/vulnerability research.
- Fix a networking issue on Kali (lost internet access).

### What I did
- Picked 3 services from the Nmap scan to investigate: MySQL 5.0.51a (3306), PostgreSQL 8.3.0-8.3.7 (5432), ProFTPD 1.3.1 (2121).
- Used `searchsploit` on each, then learned to cross-check the exact affected version against each result instead of trusting the title alone — several results didn't actually apply to the installed version.
- Confirmed a real match for MySQL (DoS, versions prior to 5.1.49) via a public advisory.
- Identified PostgreSQL's UDF command execution path as the most interesting lead, but noted it requires authenticated DB access first — not a pre-auth exploit.
- Used Metasploit's `auxiliary/scanner/postgres/postgres_login` module to test for default/weak credentials on PostgreSQL: found a valid account and got an interactive session.
- Learned the session's command syntax (`query <SQL>` instead of raw SQL), and reviewed the list of follow-up modules available from that session (hashdump, schemadump, version, etc.).
- Lost internet access on Kali; diagnosed it as eth0 (NAT) being disconnected while eth1 (lab network) still worked. Renamed the attackers' internal network from `net-attaquants` to `net-attackers` (updated identically in pfSense).
- Learned how Linux picks which interface to use per destination: longest-prefix-match routing, with `ipv4.never-default` on eth1 preventing it from competing with eth0 for the default route.
- Hit transient duplicate/corrupted ping replies after repeatedly toggling the virtual cable — resolved by reconnecting properly and flushing the ARP cache; confirmed real traffic (HTTP to Metasploitable) was unaffected.

### Problems & fixes
- `searchsploit <product>` without a version returns every version's exploits — must filter manually or re-run with the exact version string.
- Kali's eth0 showed as `disconnected` because the virtual cable was unplugged in VirtualBox from an earlier test; reconnecting it and/or recreating the nmcli profile fixed it.
- Flapping the virtual cable a few times left stale ARP entries, producing corrupted/duplicate ping replies — not an actual network fault.

### What I learned
- An exploit title's version range (`< X`) only means the bug was *introduced* at some point before X and fixed in X — it doesn't mean every older version is affected. Always check the advisory text.
- Credential enumeration (testing default/weak logins) is a standard, necessary step before any further access — not something to skip by just looking up documented defaults.
- A multi-homed Linux box (two NICs) uses the most specific matching route for each destination; the "default route" only handles traffic with no more specific match.

### Next
- Decide whether to pursue the PostgreSQL privilege-escalation path further or move to Blue Team work (Wazuh) with what's already found.
- Keep consolidating findings (service, version, port, credentials found, method used) toward the final report.

---

## Day 3 — 2026-11-20

### Goals
- Obtain an initial foothold on Metasploitable2 by targeting an unexploited service, concluding the Red Team phase before starting Blue Team detection.

### What I did
- Having already exploited PostgreSQL via UDF on Day 2, and having previously practiced exploiting services like MySQL and ProFTPD in prior lab contexts, I chose to skip redundant testing on those and focus my efforts on unfamiliar territory.
- Exploited UnrealIRCd 3.2.8.1 on Metasploitable's IRC service (port 6667) using Metasploit's `exploit/unix/irc/unreal_ircd_3281_backdoor` module — [CVE-2010-2075](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2010-2075).
- Obtained an interactive Meterpreter session; command `getuid` confirmed the session executes with `root` privileges.

### What I learned
- CVE-2010-2075 is a supply-chain backdoor: the official UnrealIRCd 3.2.8.1 source archive was tampered with at the distribution server, so the backdoor is present regardless of how the service is configured — not a logic flaw in the protocol itself.
- Landing directly as `root` reflects how permissively the service was run on Metasploitable, which is realistic for this intentionally vulnerable target but not for a hardened production system.

### Next
- Expand the assessment to web applications exposed on port 80 (DVWA).
- Document all exploitation findings and prepare for Blue Team detection (Wazuh SIEM).

---

## Day 4 — 2026-11-21

### Goals
- Assess web applications hosted on Metasploitable2 (Apache 2.2.8 / PHP 5.2.4), focusing on DVWA (Damn Vulnerable Web Application).
- Conduct web directory reconnaissance and user enumeration.
- Evaluate authentication brute-forcing tools (Burp Suite Intruder vs. THC-Hydra vs. Wfuzz).
- Test OS Command Injection across different security levels (Low, Medium, High).

### What I did
- **Directory Indexing & User Enumeration**:
  - Navigated to `http://10.0.10.30/dvwa/hackable/` and discovered directory browsing was enabled on Apache.
  - Enumerated `/dvwa/hackable/users/`, which exposed avatar image filenames revealing 5 valid application usernames: `1337`, `admin`, `gordonb`, `pablo`, and `smithy`.
- **Authentication Brute-Forcing**:
  - Intercepted the DVWA login POST request (`/dvwa/login.php`) via Burp Suite and configured Intruder using a targeted dictionary.
  - Utilized `Grep - Match` against the failure string `"Username and/or password incorrect."` to detect login anomalies; successfully identified the valid administrative credential `admin:password` (reflected by unchecked error match and distinct response length 4985 vs. 4923 bytes).
  - Attempted command-line brute-forcing using **THC-Hydra** (`hydra -l admin -P ... http-post-form`), but found it to be an **unoptimal solution** due to rigid POST-form delimiter syntax, complex cookie handling, and false positive matches (flagging 16 false matches within 2 seconds due to an incorrect failure match string and unauthenticated session redirection).
  - Switched to **Wfuzz** as an optimal alternative: configured the session cookie (`PHPSESSID` and `security=low`) with the `darkweb2017_top-1000.txt` wordlist (`wfuzz -c -z file,... -d "username=admin&password=FUZZ&Login=Login" -b "PHPSESSID=...; security=low" http://10.0.10.30/dvwa/vulnerabilities/brute/`).
  - Wfuzz successfully isolated `password` (ID 4, length 4637 characters, distinct word/line count) from incorrect attempts before hitting the server's 90-second socket timeout.
- **OS Command Injection (DVWA Ping utility `/dvwa/vulnerabilities/exec/`)**:
  - **Security = Low**: Input validation was completely absent (`shell_exec("ping -c 3 " . $target)`). Injected **Payload 1**: `10.0.10.30 && cat /etc/passwd`, which appended the command after the ping and dumped the entire `/etc/passwd` file.
  - **Security = Medium**: The application introduced a blacklist attempting to strip `;` and `&&` (`str_replace(array("&", ";"), "", $target)`). Injected **Payload 2**: `10.0.10.30 | cat /etc/passwd`, bypassing the filter via the pipe (`|`) operator and executing arbitrary commands.
  - **Security = High**: Inspected the PHP source code to assess exploitability. Identified strict input validation: the code splits the IP address by `.` (`explode(".", $target)`), verifies that exactly 4 octets exist (`sizeof($octet) == 4`), and validates that every octet is strictly numeric (`is_numeric($octet[0])` through `is_numeric($octet[3])`). Because no shell metacharacters or alphanumeric command strings can satisfy this condition, **Security = High is impossible to break**.

### Problems & fixes
- **Hydra Syntax & False Positives**: Hydra failed initially with invalid parameter count, and then generated 16 false positives because the error string pattern and cookie state were desynchronized with DVWA's session flow. Fixed by transitioning to Wfuzz, which provided cleaner HTTP header injection and accurate response filtering.
- **Wfuzz Pycurl Timeout**: Long-running brute-force requests hit a 90-second server timeout due to Apache keep-alive exhaustion on Metasploitable2. Mitigated by filtering directly on response word/character counts and using targeted password lists.
- **Medium Security Blacklist Filter**: DVWA Medium sanitized `&&` and `;`, breaking standard command chaining. Fixed by switching to the pipe operator (`|`), demonstrating the fundamental weakness of naive blacklist sanitization.

### What I learned
- Directory indexing is an overlooked high-impact information disclosure flaw that provides attackers with targeted usernames for offline or online brute-force attacks.
- Blacklisting specific characters (`&&`, `;`) in input validation is fragile and ineffective because alternative shell operators (`|`, `||`, newline, backticks) can achieve the same command chaining.
- Whitelisting and type checking (as implemented in DVWA High with `explode()` and `is_numeric()`) provide genuine security by ensuring inputs conform strictly to expected data types, making command injection impossible.
- Tool selection matters: while Burp Intruder excels in interactive GUI analysis and Wfuzz offers robust and flexible header/cookie CLI fuzzing, Hydra can be prone to false positives when web application sessions require dynamic cookie management.

### Next
- Finalize and compile all Red Team findings into `reports/pentest-report.md`.
- Begin the Blue Team phase: deploy and configure Wazuh SIEM to detect the attacks executed across Days 1–4.

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
