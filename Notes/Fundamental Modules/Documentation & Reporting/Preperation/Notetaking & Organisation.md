## Notetaking and Organisation

Your notes are the raw input to your report. The quality of your report is directly bounded by the quality of your notes. No amount of technical skill at the reporting stage recovers poorly kept notes from the testing stage.

***

## The Note Structure

The following twelve categories cover everything you need to track during any assessment. Not all will apply to every engagement type, but having them pre-built as a template means you never forget a category under pressure.

```
1. Attack Path
   Outline the full compromise path as you build it.
   Include screenshots and command output inline as you go.
   This way, the attack chain section of your report is mostly pre-written
   by the time testing ends.
   Example entry:
     "SQLi on /login -> www-data shell on WEB01
      -> sudo vim -> root on WEB01
      -> config.php creds -> dbadmin on DBSRV01
      -> Kerberoast -> DA hash cracked -> DC01 Domain Admin"

2. Credentials
   Centralised location for everything you have compromised or found.
   Format:
     Username        | Password / Hash              | Source            | System
     ----------------|------------------------------|-------------------|----------
     dbadmin         | S3cur3P@ssw0rd!              | config.php        | DBSRV01
     Administrator   | aad3b435b...:32ed87bdb5fd... | secretsdump       | DC01
     svc_backup      | $krb5tgs$23$...              | Kerberoast        | Domain
   Store this in an encrypted file or password manager, not plaintext.

3. Findings (one sub-folder per finding)
   Inside each finding folder:
     narrative.md         <- write-up of the finding
     screenshots/         <- evidence images
     commands.txt         <- exact commands and output
     repro_steps.md       <- numbered reproduction steps
   Also maintain a master findings index:
     ID  | Title                          | Severity | Host        | Status
     ----|--------------------------------|----------|-------------|----------
     F01 | SQLi on /login                 | Critical | WEB01       | Confirmed
     F02 | Sudo misconfiguration          | High     | WEB01       | Confirmed
     F03 | Credential reuse (dbadmin)     | High     | DBSRV01     | Confirmed
     F04 | Kerberoastable service account | High     | Domain      | Confirmed
     F05 | SMB signing not required       | Medium   | All hosts   | Confirmed

4. Vulnerability Scan Research
   What you scanned, what you tried, what you ruled out.
   Prevents repeating work you already did on a long engagement:
     "Ran Nessus credentialed scan against 10.10.10.0/24 on 04/04/2026.
      Reviewed all High/Critical. F01 and F02 confirmed manually.
      Nessus false positive: CVE-2021-XXXXX - service actually patched,
      banner not updated. Ruled out."

5. Service Enumeration Research
   Per service: what was found, what was tried, result.
     "Port 2121/tcp - FTP. Banner: vsftpd 3.0.3.
      Anonymous login: denied.
      CVE-2011-2523 backdoor: not applicable (version too new).
      Brute force: pending."

6. Web Application Research
   Subdomains found, applications screenshotted (EyeWitness/Aquatone),
   apps of interest, default credentials tried, CMS identified:
     "admin.target.com - Tomcat 9.0.37 - tried admin:admin, admin:password,
      tomcat:tomcat, tomcat:s3cret - s3cret worked. Finding F06."

7. AD Enumeration Research
   Step-by-step what you have and have not run:
     "BloodHound data collected 04/04 14:00.
      Ran: All collection methods.
      Kerberoastable accounts: 3 found (see credentials tab).
      ASREPRoastable: 0.
      DA path via Kerberoast confirmed in BloodHound."

8. OSINT
   Publicly found information relevant to the engagement:
     "GitHub repo: company/internal-tools - found DB password in commit
      a3f22d1 dated 2024-09-14. Password: DevDB@2024.
      Notified client at 09:32 per RoE critical finding procedure."

9. Administrative Information
   Contacts, objectives, to-do list:
     Client POC:    John Smith - j.smith@target.com - 07700 900123
     Project PM:    Jane Doe - j.doe@ourfirm.com
     Emergency POC: IT Director - 07700 900456 (for critical findings)
     To-do:
       [ ] Test VPN credentials found in OSINT against Pulse Secure
       [ ] Enumerate SNMP on 10.10.10.50 (default community string)
       [ ] Check WinRM access with compromised dbadmin creds

10. Scoping Information
    In-scope IPs, ranges, URLs, provided credentials:
      In-scope:     10.10.10.0/24, 10.10.20.0/24, admin.target.com
      Out-of-scope: 10.10.30.0/24 (production payment servers - EXPLICIT)
      VPN creds:    htb-tester / TestPass123
      AD creds:     None (blackbox)
    Keep this open during testing. Refer to it before every new scan.

11. Activity Log
    Every significant action with timestamp. This is your legal record.
      Time       | Action                              | Result
      -----------|-------------------------------------|------------------
      13:00:00   | Started Nmap sweep 10.10.10.0/24   | 14 hosts up
      13:22:14   | Full port scan WEB01 (10.10.10.5)  | 22,80,443 open
      13:44:11   | SQLi test on /login                 | Auth bypass confirmed
      14:02:33   | sudo -l as www-data                 | vim allowed as root

12. Payload Log
    Every payload deployed: what it was, where it went, hash, cleaned up?
      Time       | Host    | File path                  | Hash (SHA256) | Cleaned
      -----------|---------|----------------------------|---------------|--------
      14:38:01   | WEB01   | /tmp/shell.elf             | 3f4a8b2c...   | Yes
      15:12:44   | DBSRV01 | C:\Windows\Temp\met.exe    | 9d2e1a7f...   | Yes
      15:41:02   | DC01    | C:\Windows\Temp\chisel.exe | 2b8c4d1e...   | No - client notified
```

***

## Tmux Logging Setup

Tmux with the logging plugin is the most reliable way to capture everything you type and every output you see, regardless of whether you remembered to manually copy output. 

```
Full setup from scratch:

# Step 1: Install Tmux Plugin Manager
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

# Step 2: Create .tmux.conf
cat > ~/.tmux.conf << 'EOF'
# List of plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-logging'

# Increase scrollback buffer significantly
set -g history-limit 50000

# Initialize TMUX plugin manager (keep at bottom)
run '~/.tmux/plugins/tpm/tpm'
EOF

# Step 3: Apply the config
tmux source ~/.tmux.conf

# Step 4: Start a named session for the engagement
tmux new -s ACME_IPT

# Step 5: Install plugins (inside Tmux session)
Ctrl+B then Shift+I
# Wait ~5 seconds for installation

# Step 6: Start logging
Ctrl+B then Shift+P
# Bottom of window confirms: "Logging started. Output file: ~/tmux-YYYYMMDD.log"

# Step 7: Stop logging
Ctrl+B then Shift+P again
# OR: type exit to kill the session

Key bindings summary:
  Ctrl+B  Shift+P       -> Start / stop logging current pane
  Ctrl+B  Alt+Shift+P   -> Retroactive log dump (saves scrollback buffer)
  Ctrl+B  Alt+P         -> Screen capture of current pane (clean, no overlap)
  Ctrl+B  Alt+C         -> Clear pane history

Recommended additional plugins to add to .tmux.conf:
  set -g @plugin 'tmux-plugins/tmux-resurrect'    # restore sessions after reboot
  set -g @plugin 'tmux-plugins/tmux-sessionist'   # session management
  set -g @plugin 'tmux-plugins/tmux-pain-control' # better pane navigation

Changing default log path (add to .tmux.conf):
  set -g @logging-path "$HOME/engagement_logs"
```

### Why the Scrollback Buffer Matters

```
If you forget to start logging at session start:
  Use retroactive logging: Ctrl+B then Alt+Shift+P
  This saves whatever is in the scrollback buffer.

Default scrollback buffer is often only 2000 lines.
If testing has been ongoing for hours, you will lose most of it.
Setting history-limit 50000 in .tmux.conf prevents this.
Add it before your first real engagement, not after you need it.

Split pane logging:
  When you have two panes (e.g., Responder + ntlmrelayx):
  Trying to copy text from one pane grabs text from both = messy output.
  Solution: Ctrl+B then Alt+P takes a clean capture of the active pane only.
  Output goes to a clean file with no cross-pane contamination.
```

***

## Folder Structure

Create this structure at the start of every engagement from a script. Consistency means you never waste time looking for evidence. 

```
# One-liner to create the full structure:
mkdir -p ACME-IPT/{Admin,Deliverables,Evidence/{Findings,Scans/{Vuln,Service,Web,'AD Enumeration'},Notes,OSINT,Wireless,'Logging output','Misc Files'},Retest}

Resulting tree:
ACME-IPT/
├── Admin/                    <- SoW, kickoff notes, status reports,
│                                vulnerability notifications, RoE
├── Deliverables/             <- Report drafts and final versions
├── Evidence/
│   ├── Findings/             <- One subfolder per finding (F01, F02, etc.)
│   ├── Scans/
│   │   ├── Vuln/             <- Nessus, OpenVAS exports
│   │   ├── Service/          <- Nmap, Masscan, Rustscan output
│   │   ├── Web/              <- Burp project, ZAP, EyeWitness, Aquatone
│   │   └── AD Enumeration/   <- BloodHound JSON, CrackMapExec logs,
│   │                            PowerView CSV, Snaffler output, Impacket
│   ├── Notes/                <- Your 12-category note structure
│   ├── OSINT/                <- Maltego output, Intelx exports, screenshots
│   ├── Wireless/             <- Only if wireless is in scope
│   ├── Logging output/       <- Tmux logs, Metasploit logs
│   └── Misc Files/           <- Web shells, payloads, custom scripts
└── Retest/                   <- Mirror structure for post-remediation testing
                                 Keep retest evidence separate from original

Obsidian integration:
  Open the ACME-IPT folder as an Obsidian vault.
  Your Notes/ subfolder becomes your linked Markdown notes.
  Findings/ subfolder contains one .md file per finding.
  Everything accessible from both the CLI and the Obsidian GUI.
  Fully portable and exportable to any other Markdown-compatible tool.
```

***

## Evidence Standards

```
Screenshots:
  Always include in frame:
    - Terminal prompt (shows hostname and username)
    - The command being run
    - The relevant output
    - Date/time if not visible in terminal prompt

  Annotations (use Flameshot on Linux, Greenshot on Windows):
    - Red box around the critical finding in the output
    - Arrow pointing to the relevant field
    - Do not over-annotate - one clear highlight is better than five

  Cropping:
    Crop to show only what is relevant.
    A full 1920x1080 screenshot with one relevant line buried in it
    forces the reader to hunt for the evidence.

  File naming:
    YYYYMMDD_HHMMSS_hostname_finding.png
    20260404_144211_WEB01_sqli_auth_bypass.png

Redaction (critical - never skip this):
  What must be redacted in screenshots:
    - Full passwords (replace with <REDACTED>)
    - Full password hashes (keep first 4 and last 4 chars, replace middle)
    - PII: full names, addresses, national insurance/SSN, DOB
    - Payment card data
    - Patient health data

  Correct redaction technique:
    Use a solid black filled rectangle directly on the image.
    Edit the image file itself, not a shape overlay in Word.
    A shape overlay in Word can be deleted by anyone with the document.

  NEVER use blurring or pixelation to redact:
    Research (BishopFox Unredacter tool) shows pixelation can be reversed.
    Blurring is similarly reversible for short text.
    Only solid black bars are safe redaction.

Terminal output (preferred over screenshots where possible):
  Advantages over screenshots:
    - Client can copy/paste commands directly to reproduce
    - No encoding issues with apostrophes or quote characters
    - Easier to redact with <REDACTED> placeholder
    - Much smaller file size in the final report document
    - Easier to apply colour highlighting (blue for command, red for finding)

  Redacting hashes in terminal output:
    Before: aad3b435b51404eeaad3b435b51404ee:32ed87bdb5fdc5e9cba88547376818d4
    After:  aad3b435b51404ee[...REDACTED...]:32ed87b[...REDACTED...]376818d4

  Important: Never alter terminal output.
    It is acceptable to shorten long output and mark it:
    "[...output truncated for brevity...]"
    Or:
    "<SNIP>"
    It is never acceptable to edit the actual command or result.
    If pasting into Word, always paste as plain text (Ctrl+Shift+V)
    to strip formatting and avoid non-UTF-8 quote characters.
    Embedded formatting characters will cause client commands to fail
    when they try to reproduce your steps.
```

***

## What Not to Archive

```
Do NOT retain or exfiltrate:
  - Actual PII files (open and screenshot the directory, not the file contents)
  - Payment card data (document the exposure, not the data itself)
  - Patient health records
  - Anything that would create GDPR or compliance obligations for your firm
  - Anything described as legally discoverable in the client's industry

Correct approach for sensitive files on shares:
  Wrong:  Open the HR spreadsheet, screenshot Social Security numbers
  Right:  Screenshot the directory listing showing the file name
          Note: "HR_Employee_Data_2024.xlsx found on FILESRV01\HR_Share\"
          The client will understand what is in a file named that.

If you have already collected sensitive data inadvertently:
  Notify your manager immediately
  Document what was collected and how
  Delete it from your systems as soon as practically possible
  Note this in the report appendices

Why this matters beyond ethics:
  GDPR (UK and EU) creates obligations for any firm that processes
  personal data about EU/UK data subjects.
  If your testing laptop holding real PII is stolen or breached,
  your firm may have notification obligations to the ICO (Information
  Commissioner's Office in the UK) within 72 hours.
  The simplest compliance position: do not have the data in the first place.
```

***

## Account Creation and System Modifications

```
Every change you make to a client system must be tracked with:
  - IP address / hostname of the affected system
  - Timestamp of the change
  - Description of what was changed
  - Location on the host (file path, registry key, service name)
  - Application or service affected
  - Account name and password if an account was created

Minimum approval required before making changes:
  Get written confirmation (email is sufficient) before:
  - Creating any user accounts
  - Modifying service configurations
  - Adding firewall rules or GPO settings
  - Changing any running service
  - Making any registry modification that is not part of a test payload

This is typically agreed during the kickoff call:
  "What is your threshold for system modification beyond which
   you want us to notify you before proceeding?"
  Document the answer in your Admin notes and the RoE.

Format for tracking in your Payload Log:
  Time       | Host    | Change type   | Details                  | Reverted
  -----------|---------|---------------|--------------------------|----------
  15:03:11   | WEB01   | Account added | user: backdoor           | Yes 16:22
  15:41:22   | FILESRV | File uploaded | C:\Temp\chisel.exe       | Yes 17:00
  16:02:44   | DC01    | Reg key added | HKCU\Run\Update          | No - notify client
```
