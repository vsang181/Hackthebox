## Documentation and Reporting

Strong documentation is not a soft skill that complements technical ability. It is a core professional competency that directly determines whether your findings drive real change, whether you are legally protected when things go wrong, and whether you have a career in consulting long-term.

***

## Why Documentation Is a Career-Defining Skill

The three real-world scenarios in the module illustrate something most new testers learn the hard way: your documentation is not just for the report. It is your legal defence, your professional reputation, and your safety net.

```
Scenario 1 - VM failure during month-long engagement:
  What saved it:   Daily backups to shared storage, detailed notes in OneNote
                   on base workstation (not only on the testing VM)
  What would have happened without it:
                   Weeks of work lost, client deliverable missed,
                   potential loss of client and employment
  Lesson:          Never store notes only on the testing VM.
                   Sync evidence to a separate encrypted location daily.

Scenario 2 - Accidental scan of out-of-scope critical servers:
  What saved it:   Timestamped raw scan data, signed scope confirmation from client,
                   documented IP ranges that matched the agreed scope
  What would have happened without it:
                   No way to prove the IPs were in the given scope.
                   Blame lands entirely on the tester.
  Lesson learned:  After this, explicitly ask clients for a list of hosts
                   to exclude, even if they were not in scope.
                   Document that exclusion list in the RoE.

Scenario 3 - Network slowdown blamed on tester:
  What saved it:   Scan output proving nothing aggressive was running.
                   Documentation forced the client to investigate further.
                   Root cause found: debug mode enabled on all network devices.
  What would have happened without it:
                   Source IPs blocked, engagement cancelled, reputation damaged,
                   blame permanently attributed to the tester.
  Lesson:          Always export and timestamp all scan output in real time.
                   Not just the findings - the full raw tool output.

Common thread across all three:
  In every case, documentation either saved the tester or created
  a lesson that improved their process.
  In none of these cases was technical skill the deciding factor.
  It was organisation, process, and discipline.
```

***

## The Snapshot-in-Time Principle

Every penetration test report must clearly communicate that it reflects a specific window of time and nothing outside that window.

```
Why this matters:
  A vulnerability patched the week after testing ends is not in the report.
  A new system added to the network post-testing is not covered.
  A finding remediated mid-test may still appear in the report.
  Without this disclaimer, clients may hold you responsible for
  vulnerabilities discovered by a real attacker months later.

Required language in every report (scope/methodology section):
  "All testing activities were performed between [start date] and [end date].
   This report represents a snapshot in time during the aforementioned testing
   period and [Firm Name] cannot attest to the state of any client-owned
   information assets outside of this testing window."

Additional context to include in the overview section:
  - Type of assessment (internal, external, web app, AD, etc.)
  - Names or roles of testers who performed the work
  - Source IP addresses used during testing
  - Testing method (on-site, remote over VPN, from client-hosted VM)
  - Any special considerations or limitations encountered
  - Explicit statement of what was NOT tested and why
```

***

## Core Documentation Principles

There is no universal format, but there are non-negotiable fundamentals that apply regardless of client, engagement type, or firm.

```
Principle 1: Document everything in real time
  Never rely on memory to reconstruct notes after the fact.
  By the time you are writing the report, the nuance of
  what you observed and why you made each decision is gone.
  Terminal logging is not optional, it is the baseline:
  script -a ~/engagement/sessions/$(date +%Y%m%d_%H%M%S).log

Principle 2: Evidence must be self-contained
  Any screenshot or piece of evidence must prove its own context.
  If someone reads your report without talking to you, they must
  be able to look at any piece of evidence and understand:
  - What system this was on (hostname visible)
  - What user was running the command (whoami or prompt visible)
  - What network address this system was at (IP visible)
  - What was being demonstrated (action visible)
  A screenshot that shows only a password hash with no other context
  proves nothing and will be challenged.

Principle 3: Separate your note types
  Raw notes:          Everything, messy, timestamped, unedited
  Findings log:       Structured list of confirmed vulnerabilities
  Activity log:       Chronological record of every action taken
  Evidence folder:    Screenshots and tool output per finding
  Report draft:       Clean, client-facing write-up

Principle 4: Back up daily, never only on the test VM
  Test VM can crash, become corrupted, or be accidentally rolled back.
  Evidence must exist in at least two locations at all times:
  - Test VM (working copy)
  - Encrypted share or local workstation (backup copy)

Principle 5: Never store real sensitive data long-term
  Credentials, hashes, PII, payment data encountered during testing
  must not persist on your systems after the engagement closes.
  Store finding impact descriptions, not the actual data itself.
```

***

## Documentation Infrastructure Setup

Before any engagement begins, your documentation environment should be ready. Setting it up during the engagement wastes time and creates gaps.

```
Recommended folder structure (create from a template VM):

/engagement_clientname_YYYYMM/
  00_admin/
    scope.txt                  <- in-scope IPs, URLs, systems
    roe_signed.pdf             <- signed Rules of Engagement
    kickoff_notes.txt          <- notes from kickoff call
    contacts.txt               <- client emergency contacts

  01_recon/
    nmap/
      initial_sweep.txt
      full_tcp.txt
      udp_top100.txt
      targeted_scripts.txt
    osint/
    dns/
    screenshots/

  02_exploitation/
    finding_001_sqli/
      request.txt
      response.txt
      commands.txt
      screenshots/
    finding_002_sudo_lpe/
      (same structure)

  03_post_exploitation/
    credentials.txt             <- encrypted, password manager preferred
    hashes.txt
    network_map.png
    bloodhound/
    pillaging/

  04_lateral_movement/
    pivot_setup.txt
    hosts_compromised.txt
    (per-host subfolders)

  05_evidence/
    all_screenshots/            <- copies of all screenshots in one place
    tool_output/                <- raw output from all tools run

  06_report/
    draft_v1.docx
    draft_v2_client_comments.docx
    final.docx
    post_remediation_v1.docx

  07_logs/
    session_20260404_133000.log
    session_20260404_150000.log

Note-taking tool recommendation:
  Obsidian:     Local, markdown-based, no cloud sync of sensitive data,
                linkable notes, works well for structuring findings
                HTB Academy provides a sample Obsidian notebook in the
                module resources tab

  CherryTree:   Hierarchical, encrypted local notebooks, good for
                engagements that need a single portable file

  Avoid:        Cloud-synced tools for sensitive client data unless
                your firm explicitly approves and the sync is encrypted
```

***

## The Two Documentation Types Explained

Every engagement produces two distinct outputs aimed at two completely different audiences. Both must be written from the same set of notes.

```
Technical Documentation:
  Audience:  Sysadmins, developers, security engineers
  Purpose:   Enable exact reproduction and remediation of each finding
  Language:  Precise, uses technical terms correctly
  Includes:  Exact commands, payloads, URLs, port numbers, version numbers,
             CVE references, CWE classifications, step-by-step reproduction

  Test:      Could a developer who was not on the call follow these steps
             and reproduce the finding without asking you a single question?
             If no: not detailed enough.

Non-Technical Documentation (Executive Summary):
  Audience:  CEO, CFO, board members, legal counsel, risk officers
  Purpose:   Communicate business risk and required investment
  Language:  Plain English, zero jargon, business impact focused
  Includes:  What was tested, what was found, what could happen if
             nothing is fixed, what needs to happen to address it

  Test:      Could your parent read this and understand what is at risk?
             If they would stop at a technical term and be confused:
             rewrite it.

Where most testers fail:
  They write the executive summary like a technical finding.
  They write technical findings at a surface level that lacks reproduction detail.
  Both sections get written for the tester's own understanding
  rather than for the reader's needs.
```

***

## Building Good Documentation Habits Through Practice

The module recommends practising against training targets before using these skills on a real engagement. The loop below applies to every module, machine, or lab you work through.

```
Practice documentation loop:

During the exercise:
  1. Enable terminal logging before your first command
  2. Take a screenshot at every meaningful discovery
     (open port, version found, auth bypass, privilege escalation)
  3. Name screenshots immediately: YYYYMMDD_host_finding.png
  4. Keep a running findings log with severity, title, and host
  5. Keep a running activity log with timestamps

After the exercise:
  6. Write a technical finding write-up for each confirmed vulnerability
     - Title, severity, description, evidence, reproduction steps, remediation
  7. Write an executive summary paragraph describing the overall result
  8. Review both against the checklist:
     - Is every screenshot self-contained?
     - Are reproduction steps specific enough to follow without help?
     - Is the executive summary free of jargon?
     - Does the remediation address root cause?
  9. If using retired machines: compare against write-ups and IppSec videos
     - What did you document well?
     - What did you miss?
     - What would a client have needed that your notes do not provide?

The goal of this loop:
  Make documentation a parallel habit during testing, not a separate
  task done at the end.
  When documentation is treated as an afterthought, it shows in the report.
  When it is built into your workflow as a continuous process,
  the report practically writes itself.
```

***

## What Separates a Good Report From a Great One

```
Good report:
  - All findings included
  - Evidence present for each
  - Reproduction steps are mostly followable
  - Executive summary exists

Great report:
  - Attack chain narrative shows full impact of chained vulnerabilities
  - Executive summary drives urgency without causing panic
  - Reproduction steps so precise that remediation can be confirmed
    without calling the tester
  - Remediation advice tied to root cause with authoritative references
  - Near / medium / long-term roadmap gives the client a prioritised plan
  - Appendices provide complete raw data for the client's technical team
  - Clearly distinguishes between what was confirmed exploitable and
    what was out of scope or not tested
  - Includes the snapshot-in-time disclaimer
  - DRAFT / FINAL version control clearly applied

The final measure:
  A great report makes a good impression without the tester being in
  the room. Most of the people who read it will never meet you.
  The report is your professional identity for that client.
  Treat it accordingly.
```
