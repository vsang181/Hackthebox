## Post-Engagement

Post-engagement covers everything that happens after your last command runs against a target. It is not an afterthought. It is a phase with contractual, legal, and professional obligations that define how you are perceived as a consultant long after the technical work is done.

***

## Cleanup

Every artefact you leave on a client system is a liability for both you and the client. Thorough cleanup is non-negotiable.

```
What to remove:

Tools and binaries uploaded:
  - Any exploit binaries, msfvenom payloads, shells
  - Enumeration tools: winPEAS, linPEAS, SharpHound, mimikatz
  - Pivoting tools: Chisel, socat, netcat if not native
  - Any files staged in /tmp, C:\Windows\Temp, C:\Users\*\AppData

Accounts created:
  # Windows - remove backdoor local accounts:
  net user <backdoor_account> /delete

  # Linux - remove created accounts:
  userdel -r <backdoor_account>

Persistence mechanisms:
  # Remove scheduled tasks:
  schtasks /delete /tn "TaskName" /f

  # Remove registry run keys:
  reg delete "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "KeyName" /f

  # Remove cron entries:
  crontab -e   <- remove added lines

  # Remove SSH authorised keys added:
  # Edit ~/.ssh/authorized_keys and remove your key

  # Remove added services:
  sc stop ServiceName && sc delete ServiceName

Configuration changes:
  # If you modified any settings (firewall rules, registry values,
  # service configs) to facilitate access, revert them
  # Cross-reference your notes - every change made should be logged

What to do if you cannot access a system to clean up:
  - Document the specific artefact, its location, and the system it is on
  - Alert the client directly (do not wait for the report)
  - Include it in the report appendices
  - Provide exact removal instructions so their team can clean it up

Cleanup documentation (include in report appendix):
  Even if you successfully removed everything, document:
  - What was uploaded, where, and when
  - What accounts were created and when deleted
  - What configuration changes were made and when reverted
  - This gives the client context if alerts fire relating to your activity
```

***

## Documentation and Reporting

The report is your primary deliverable and the only tangible output the client retains after the engagement ends. Quality here reflects directly on you and your firm.

```
Before disconnecting from client environment, confirm you have:

Evidence per finding:
  - Screenshot showing exploitation (hostname + user + IP visible)
  - Command output with timestamps
  - Scan results (Nmap, Nessus, etc.) in raw and parsed format
  - Burp Suite project file (for web assessments)
  - BloodHound data export (for AD assessments)
  - Password cracking results and hash lists (redacted for report)
  - Network capture files if relevant
  - List of all compromised hosts and accounts
  - List of all files transferred to/from client systems

Data you must NOT retain after the engagement:
  - Real PII (names, addresses, national insurance/SSN numbers)
  - Real payment card data
  - Real health records
  - Any sensitive customer data encountered during pillaging
  These should be documented by category and impact only, not copied

Report structure:

1. Attack Chain (if full compromise achieved):
   Step-by-step narrative from initial access to highest privilege
   Each step: method used, evidence, impact at that point

2. Executive Summary:
   Written for a non-technical audience (CEO, board, legal)
   No jargon. Focus on: what was tested, what was found, what is the risk,
   what happens if it is not fixed
   One page maximum - executives do not read further

3. Findings (one section per finding):
   - Finding title
   - Risk rating (Critical / High / Medium / Low / Informational)
   - CVSS score (where applicable)
   - Affected hosts / systems
   - Vulnerability description
   - Business impact
   - Evidence (screenshots, output)
   - Reproduction steps (exact steps, commands, payloads used)
   - Remediation recommendation (address root cause)
   - External references (CVE, CWE, OWASP, vendor advisory)

4. Near / Medium / Long-term Recommendations:
   Near-term:   Patch critical findings, change compromised passwords
   Medium-term: Implement missing controls (MFA, SMB signing, logging)
   Long-term:   Mature the security programme (patch management, awareness
                training, security architecture review)

5. Appendices:
   A. Scope (in-scope IPs, URLs, time windows)
   B. OSINT findings (if in scope)
   C. Discovered open ports and services (full Nmap output)
   D. Compromised hosts and accounts list
   E. Files transferred to client systems during testing
   F. Account creations and configuration changes (and cleanup confirmation)
   G. Password cracking analysis (wordlists, rules, time taken, success rates)
   H. Active Directory security analysis (BloodHound summary if applicable)
   I. Supplementary scan data and tool output
```

***

## Risk Rating Methodology

Be consistent across all findings and all engagements.

```
Critical:
  - Direct path to domain/full compromise
  - Unauthenticated RCE on internet-facing system
  - Authentication bypass on critical application
  - Examples: EternalBlue on internet-facing host, SQLi with OS command exec

High:
  - Significant impact requiring some conditions
  - Authenticated RCE, sensitive data exposure, privilege escalation
  - Examples: Kerberoastable service account with weak password,
              SQL injection returning sensitive data, LPE to SYSTEM

Medium:
  - Limited direct impact, often requires chaining
  - Examples: SMB signing disabled, stored XSS, outdated software
              without confirmed working exploit, default credentials
              on non-critical internal system

Low:
  - Minimal impact, informational value
  - Examples: Directory listing enabled, missing security headers,
              verbose error messages, weak TLS cipher suites

Informational:
  - Not a vulnerability, but worth the client knowing
  - Examples: Use of deprecated but not yet vulnerable software version,
              internal hostname disclosure, non-sensitive data in robots.txt

CVSS context note:
  Your risk rating should factor in the client's environment, not just
  the CVSS score in isolation. A medium CVSS finding on an internet-facing
  system in a PCI-DSS scope may be rated High in your report.
  Always contextualise.
```

***

## Report Lifecycle

```
Draft -> Review -> Final -> Post-Remediation

1. Draft report delivered:
   - Clearly watermarked DRAFT on every page
   - Delivered within agreed timeframe (typically 5-10 business days
     after testing ends)
   - Delivered via encrypted channel (encrypted email, secure portal)
   - Never sent as unencrypted email attachment

2. Client review period:
   - Client distributes internally to technical leads and management
   - Client compiles questions, disputes, and management responses
   - Typical review period: 1-2 weeks

3. Report review meeting:
   - Walk through each finding verbally
   - Do not read the report word for word - add context from your experience
   - Allow client to ask questions and request clarifications
   - Note any factual corrections needed
   - Drive home the root cause message for key findings

4. Final report issued:
   - Incorporate any agreed corrections
   - Update DRAFT watermark to FINAL
   - Some compliance frameworks (PCI-DSS QSA audits) only accept FINAL designation
   - Reissue via the same secure channel

5. Post-remediation report:
   - Client provides evidence of fixes or a list of remediated findings
   - Retest each finding against the live environment
   - Issue a new report showing before/after status per finding:

   Finding                          Before          After
   SQL Injection                    High            Remediated
   SMB Signing Not Required         Medium          Remediated
   Inadequate Egress Filtering      High            Not Remediated
   Default Credentials (printer)   Medium          Not Remediated
   Directory Listing Enabled        Low             Remediated

   - For each still-open finding: re-confirm exploitation still works
   - For each remediated finding: show evidence the fix is effective
     (failed exploit attempt, updated scan showing patched version)
```

***

## Your Role in Remediation - Boundaries

```
What you should do:
  - Explain the vulnerability mechanism clearly in the report
  - Provide remediation recommendations at a conceptual level
  - Be available to clarify findings during the review meeting
  - Demonstrate the finding again if the dev team cannot reproduce it
  - Point to relevant vendor documentation, CIS benchmarks, OWASP guidance

What you must NOT do:
  - Write corrected code for a developer
  - Patch systems yourself
  - Make Active Directory configuration changes
  - Implement firewall rules or GPO changes
  - Prescribe the exact line of code that should be changed

Why this boundary matters:
  If you fix the issue and then test the fix, you are auditing your own work.
  This is a conflict of interest that invalidates the independence of the assessment.
  Any compliance auditor reviewing the pentest will scrutinise this.
  It also shifts liability - if your fix introduces a new vulnerability,
  you are now responsible.

Example of the right level of remediation advice:
  Too specific:
    "Change line 47 of login.php to:
     $stmt = $pdo->prepare('SELECT * FROM users WHERE username = ?');
     $stmt->execute([$username]);"

  Correct level:
    "Remediate SQL injection by replacing dynamic query construction
     with parameterised queries or prepared statements throughout the
     application. Reference OWASP's SQL Injection Prevention Cheat Sheet
     for implementation guidance."
```

***

## Data Retention and Destruction

```
During the engagement, all data must be:
  - Stored on an encrypted volume (VeraCrypt, BitLocker, LUKS)
  - On firm-owned or tester-controlled infrastructure only
  - Not on personal devices, personal cloud storage, or shared drives

After the engagement closes:
  - All data wiped from tester workstations and VMs
  - Evidence retained for agreed period (typically 6-12 months)
    stored in secure, encrypted firm infrastructure
  - Retained data available on request from client or authorised parties
  - Virtual machines used for testing wiped or destroyed

PCI DSS guidance (applies broadly as best practice):
  "Retain evidence for a period of time while considering any local,
   regional, or company laws that must be followed for the retention
   of evidence. This evidence should be available upon request from
   the target entity or other authorized entities as defined in the RoE."

For post-remediation testing:
  Create a fresh VM specific to that engagement
  Do not reuse a VM from a previous client engagement
  This prevents contamination of evidence and data leakage between clients

Jurisdiction note (relevant for UK-based consultants):
  UK GDPR applies to any personal data encountered or retained
  Data minimisation principle: if you do not need it, do not keep it
  Breach notification: if tester infrastructure holding client data
  is itself compromised, obligations may exist under UK GDPR Article 33
```

***

## Close Out Checklist

```
Technical:
  [ ] All tools and payloads removed from client systems
  [ ] All created accounts deleted
  [ ] All persistence mechanisms removed
  [ ] All configuration changes reverted
  [ ] Cleanup confirmed in report appendix
  [ ] Test VMs wiped or archived encrypted

Documentation:
  [ ] Draft report delivered within agreed timeframe
  [ ] Report review meeting completed
  [ ] All client questions addressed
  [ ] Final report issued
  [ ] Post-remediation testing completed (if in scope)
  [ ] Post-remediation report issued

Administrative:
  [ ] Invoice submitted per contract terms
  [ ] All client data secured and/or destroyed per retention policy
  [ ] Post-assessment satisfaction survey sent to client
  [ ] Internal debrief completed (what went well, what to improve)
  [ ] Findings logged in firm's internal knowledge base (anonymised)

The last point matters more than most new consultants realise.
A client will remember how you communicated, how you handled problems,
and how professional you were throughout far longer than they will
remember the specific CVEs you found. Technical skill gets you in the door.
Professional conduct and communication are what generate repeat business.
```
