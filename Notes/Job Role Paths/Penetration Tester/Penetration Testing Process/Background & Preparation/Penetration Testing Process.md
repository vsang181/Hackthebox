## Penetration Testing Process

The penetration testing process is not a rigid checklist but a flexible sequence of interdependent stages that adapt to each unique environment. Every stage feeds into the next, and findings at any stage can send you back to an earlier one, making it inherently iterative rather than strictly linear.

***

## The Eight Stages

### 1. Pre-Engagement

Everything before you touch a system. This stage defines the legal and operational boundaries of the entire engagement.

```
Deliverables produced at this stage:
- Non-Disclosure Agreement (NDA)
- Statement of Work (SoW) / Contract
- Rules of Engagement (RoE)
- Scope document (in-scope and out-of-scope systems)
- Time estimation and testing windows
- Emergency contacts

Key discussions with client:
- What is the goal? (find all vulns vs reach a specific target)
- What type of test? (Black / Grey / White box)
- What perspective? (External / Internal / Both)
- What is explicitly excluded? (production DBs, fragile systems, DoS)
- What happens if something breaks during testing?
- Who needs to be notified and when?
```

### 2. Information Gathering

Active and passive reconnaissance to map the target before any exploitation is attempted.

```
Passive (no direct interaction with target):
- OSINT: LinkedIn, job postings, WHOIS, DNS records, Shodan
- Google dorking for exposed files, subdomains, login portals
- Certificate transparency logs for subdomain enumeration
- Social media for employee names, technologies, office locations

Active (direct interaction with target):
- Port scanning: nmap, masscan
- Service enumeration: nmap scripts, Netcat banner grabbing
- Web application fingerprinting: whatweb, wappalyzer
- DNS enumeration: dnsenum, ffuf with DNS wordlist
- SMTP user enumeration (if mail servers in scope)

Goal: Build a complete map of the attack surface
Output: List of hosts, services, versions, technologies, potential entry points
```

### 3. Vulnerability Assessment

Analyse what you found in information gathering against known weaknesses.

```
Methods:
- Automated: Nessus, OpenVAS, Qualys against identified hosts
- Manual: Cross-reference service versions against CVE databases (NVD, Exploit-DB)
- Web: Nikto, Burp Suite Scanner for web applications
- Windows: WinPEAS, PowerUp, Sherlock, Windows-Exploit-Suggester

Output:
- Prioritised list of potential attack vectors
- CVSS scores or internal risk rating per finding
- Decision on which vectors to test first (highest impact, most reliable)

Important distinction:
Vulnerability assessment = identifying potential weaknesses
Exploitation = proving they are actually exploitable
These are separate stages with separate outputs
```

### 4. Exploitation

Use vulnerability assessment findings to gain initial access.

```
Process:
1. Prepare exploit code, tools, listener
2. Test in isolated lab environment if possible before live execution
3. Execute against target within agreed testing window
4. Document: exact command used, timestamp, outcome
5. Obtain stable shell/foothold before moving to post-exploitation

Common outcomes:
- Low-privilege shell (web user, service account)
- Authenticated session via stolen credentials
- Code execution via vulnerable service or misconfiguration

Key principle: confirm and stabilise access before continuing
Unstable foothold = lost access = start over
```

### 5. Post-Exploitation

Once inside a system, extract value and prepare for further movement.

```
Activities in order of priority:
1. Situational awareness (whoami, hostname, ipconfig, systeminfo)
2. Privilege escalation (this entire module = post-exploitation)
3. Credential harvesting (Mimikatz, hashdump, browser creds, config files)
4. Pillaging (sensitive files, database creds, SSH keys, mRemoteNG configs)
5. Persistence (only if agreed in RoE and needed for long-term engagement)
6. Network enumeration (what else is reachable from here?)

Output:
- Credentials found
- Network segments discovered
- Sensitive data locations documented (not necessarily exfiltrated)
- Privilege escalation path documented step by step
```

### 6. Lateral Movement

Use what was found in post-exploitation to compromise additional hosts.

```
Common techniques:
- Pass-the-Hash: use NTLM hash without cracking
  impacket-psexec -hashes :<NThash> administrator@<target>
- Pass-the-Ticket: use stolen Kerberos TGT
- Credential reuse: try cracked passwords across all known hosts
- Remote services: WinRM (evil-winrm), RDP, SSH with found credentials
- SMB: psexec, smbexec, wmiexec to move to adjacent hosts

Iterative cycle:
Land on new host -> Post-exploit -> Find new creds -> Lateral move -> repeat

Tracking movement:
Maintain a diagram or spreadsheet:
  Host A (web server, low-priv user)
    -> Escalated to SYSTEM via MS16-032
    -> Found: sqlsvc password in registry
  Host B (SQL server, sqlsvc = local admin via credential reuse)
    -> Dumped SA password from SQL config
    -> Found: domain admin hash in LSASS
```

### 7. Proof-of-Concept

Document and demonstrate the full attack chain clearly enough that the client can reproduce it and understand its impact.

```
What a good PoC includes:
- Step-by-step numbered reproduction steps
- Screenshots at each critical step
- Exact commands run with full syntax
- Timestamps
- Evidence of impact (files accessed, data read, SYSTEM shell screenshot)

Format per finding:
Title: [Vulnerability Name]
Severity: Critical / High / Medium / Low
CVSS Score: X.X
Affected Asset: [hostname / IP / URL]

Description:
  [What the vulnerability is and why it exists]

Reproduction Steps:
  1. [Step one with exact command/action]
  2. [Step two with screenshot]
  ...

Impact:
  [What an attacker could achieve with this vulnerability]

Remediation:
  [Specific, actionable fix recommendation with KB/patch number if applicable]

If feasible: provide a script that automates the exploitation steps
This helps client reproduce, validate the fix, and prioritise remediation
```

### 8. Post-Engagement

Clean up, report, present, and archive.

```
Cleanup (must be done before report delivery):
- Remove all tools, payloads, and scripts uploaded to client systems
- Delete any user accounts or backdoors created during testing
- Remove scheduled tasks or persistence mechanisms added
- Confirm removal with client

Deliverables:
- Technical Report (for sysadmin/security team)
- Executive Summary (for C-level, non-technical audience)
- Findings appendix with raw evidence

Report walkthrough meeting:
- Walk client through findings one by one
- Clarify any ambiguities
- Help them understand and prioritise remediation
- Answer questions about reproduction and risk

Optional follow-on:
- Executive presentation to board or leadership
- Post-remediation retest (retest specific findings after client patches)
- Data archival per contractual obligations (typically 1-3 years)
```

***

## The Iterative Nature - Why It Is Not Linear

```
Real engagement flow vs textbook flow:

Textbook: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8

Real:
Pre-engagement -> Information Gathering
                       |
                       v
               Vulnerability Assessment
                       |
                       v
                  Exploitation -> FAIL -> back to Vuln Assessment
                       |
                      SUCCESS
                       |
                       v
               Post-Exploitation
                  |         |
                  v         v
            Escalate    Pillage creds
            privileges       |
                             v
                    Lateral Movement -> Land on new host
                             |              |
                             v              v
                         Proof of    -> Back to Post-Exploitation
                         Concept          on new host

Each new host you land on restarts the post-exploitation -> lateral movement
cycle with new context and potentially new objectives
```

***

## Process vs Checklist - The Key Mental Model

```
A checklist says: "Did you do X? Yes/No. Move on."
A process says:   "Based on what you found, what makes sense to try next?"

This is why WinPEAS output requires human interpretation.
The tool finds 50 things. The tester decides which 3 are actually exploitable
in this specific environment given this specific configuration.

The skill being developed across all HTB Academy modules is not memorising
techniques - it is building the judgment to navigate this process efficiently
under time pressure in environments you have never seen before.
```
