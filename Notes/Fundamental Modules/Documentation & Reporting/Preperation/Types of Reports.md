## Types of Reports

The report deliverable is what the client is ultimately paying for. Understanding which report type a client needs, and why, shapes everything from how you conduct the assessment to how you structure your findings.

***

## Assessment Types and Their Reports

Different assessments produce different scopes of finding, which drives different report structures.

```
Assessment Type     Testing Depth          Report Focus
-----------------   --------------------   ------------------------------------------
Vulnerability       Automated scanning,    Theme-based patterns, severity counts,
Assessment          no exploitation        false positive identification, scanner data

External Pentest    Internet-facing only   OSINT findings, attack surface, any
                                           achieved external-to-internal access

Internal Pentest    Inside the network     AD compromise path, lateral movement,
                                           privilege escalation chains, full attack chain

Web App Pentest     Application layer      OWASP-categorised findings, role-based
                                           testing results, API and business logic flaws

Purple Team         Collaborative with     Detection gap analysis, alert coverage,
                    blue team              specific TTP detection status per finding

Cloud Pentest       Cloud infrastructure   Misconfigurations, IAM abuse, secrets
                    and services           exposure, cloud-specific attack paths

IoT Assessment      Hardware, cloud,       Multi-layer findings across hardware,
                    network, application   firmware, network protocol, and cloud

Evasive/Adversary   Long-term, stealthy    Blind spots identified, detection timeline,
Simulation          threat simulation      dwell time before detection
```

***

## Vulnerability Assessment Reports

Vulnerability assessments do not involve exploitation. The report reflects scan data, validation, and pattern analysis rather than proven attack chains.

```
Key differences from a pentest report:

Scope of work:
  Run authenticated or unauthenticated scans against the environment.
  Validate scanner results (confirm version is actually vulnerable or
  confirm the misconfiguration exists) without gaining a foothold.
  The goal is not to prove exploitability but to identify potential exposure.

Report structure specifics:
  - No attack chain section (no exploitation occurred)
  - No compromised accounts or hosts appendix
  - Heavy emphasis on finding patterns and root cause themes
  - Distinguish confirmed vulnerabilities from unvalidated scanner findings
  - Distinguish true positives from false positives explicitly

Handling the volume problem:
  A credentialed internal vulnerability scan can return thousands of findings.
  Mapping them to procedural deficiencies is more valuable than
  listing every CVE individually:
  Theme: "Widespread use of end-of-life operating systems across 34 hosts"
  Theme: "Missing security patches older than 6 months on 47% of servers"
  Theme: "Default service account passwords on 12 database servers"
  These themes are actionable at a programme level, not just a host level.

Internal vs External:
  External: anonymous perspective from the internet, public-facing systems only
  Internal: behind the firewall, may be unauthenticated or with provided creds
  Credentialed internal scans produce significantly more findings
  but are more accurate and less generic than unauthenticated scans
```

***

## Penetration Test Report Types

### Internal Penetration Test

The most comprehensive report type, particularly when domain compromise is achieved.

```
Unique elements compared to other report types:

Attack chain (mandatory if AD or full internal compromise achieved):
  Full narrative from initial foothold to highest achieved privilege.
  Every hop documented with evidence.
  Break-the-chain analysis showing which individual fixes disrupt the path.

AD-specific appendices:
  BloodHound summary (key attack paths visualised)
  Domain enumeration data (privileged accounts, stale accounts, misconfigs)
  Password policy analysis
  Kerberoastable / ASREPRoastable account counts
  DCSync / replication rights holders

Compromised accounts list:
  Every account accessed or whose credentials were obtained.
  Marked as Active Directory, local, or service account.

Testing perspectives available:
  Black box:  Name of company and a network connection. No credentials, no info.
  Grey box:   In-scope IP ranges and CIDR blocks provided. No credentials.
  White box:  Full information - credentials, source code, configs, architecture diagrams.

Evasion levels:
  Non-evasive:   Maximum coverage, security tools may detect and block.
                 Document what was caught as evidence of working defences.

  Hybrid-evasive: Start evasive, gradually increase noise.
                  Document at what level detection occurred.
                  Recommended for clients with partial defensive maturity.
                  Once detected, client typically asks you to switch to non-evasive
                  for the remainder of the assessment.

  Fully evasive:  Remain undetected throughout.
                  Simulates advanced persistent threat.
                  Best suited for security-mature organisations.
                  Time constraints differ from real attackers - note this in the report.
```

### External Penetration Test

```
Additional elements compared to internal:

OSINT appendix:
  What was found about the company from public sources.
  Categories to document:
    Public DNS and domain ownership records
    Email addresses (and any found in breach databases)
    Subdomains discovered via passive and active enumeration
    Third-party vendors and their access to the client environment
    Similar/typosquatted domains that could be used for phishing
    Public cloud resources (S3 buckets, Azure blobs, GCP storage)
    GitHub / GitLab repositories with sensitive content
    LinkedIn-enumerated employee names and roles
    Job postings revealing internal technology stack

External attack surface section:
  All internet-facing services identified
  Which were in scope vs discovered but out of scope
  SSL/TLS configuration assessment
  Certificate expiry dates if relevant

If internal compromise achieved from external:
  Treat it like an internal report with a full attack chain.
  The entry point was external, which is significant and must be highlighted.
  External-to-internal compromise is typically a Critical finding in itself.
```

***

## The Report Lifecycle

```
Phase 1: Draft Report
  First deliverable to the client.
  Watermarked DRAFT on every page header and footer.
  Delivered within agreed timeframe after testing ends.
  Delivered via encrypted channel only.

  Client uses the draft to:
  - Distribute internally to technical leads and management
  - Add management responses to each finding (their remediation commitment)
  - Request clarifications or factual corrections
  - Adjust any language they find inaccurate or problematic

  You should expect and welcome draft feedback.
  It is part of the engagement, not a criticism of your work.

Phase 2: Report Review Meeting
  Walk through findings verbally with the client.
  Do not read the report word for word.
  Add context from your experience that is not in the written text.
  Clarify technical findings for non-technical attendees.
  Note any agreed changes.

Phase 3: Final Report
  Incorporate agreed changes from the review.
  Change DRAFT watermark to FINAL.
  Reissue via the same secure channel.
  Some compliance frameworks (PCI-DSS QSA assessments, SOC 2 audits)
  only accept FINAL designation. Never skip this step.

Phase 4: Post-Remediation Report
  Scope carefully to avoid scope creep:
    - Retest only previously reported findings
    - Retest only the originally affected hosts
    - Set a time limit on when remediation testing can be requested
    - Do not run new large-scale scans (you will find new things and
      the scope will expand beyond control)

  What happens without these boundaries:
    - Client requests retest a year later, environment has changed entirely
    - New hosts affected by old findings discovered, endless loop begins
    - Pressure to modify severity ratings or close findings prematurely

  How to handle pressure to modify findings inappropriately:
    Be clear about ethical boundaries without implying dishonesty.
    Offer a constructive path: many auditors accept a documented
    remediation plan with a justified timeline instead of full closure
    within the examination period. Offer this to the client.
    If retesting is delayed significantly, note it explicitly:
    "This retest was performed X months after the original assessment.
     The environment has likely changed, and only originally reported
     findings on originally reported hosts were assessed."

  Report format options for post-remediation:
    Option A: Update the original report, tag each finding with status:
              Remediated / Not Remediated / Partially Remediated
    Option B: Issue a new stand-alone report with comparison tables
              showing before and after status per finding

  Example comparison table:
    #  | Finding                    | Severity | Status
    ---|----------------------------|----------|------------------
    1  | SQL Injection              | Critical | Remediated
    2  | Default Credentials        | High     | Remediated
    3  | SMB Signing Not Required   | Medium   | Not Remediated
    4  | Directory Listing Enabled  | Low      | Not Remediated
    5  | Unrestricted File Upload   | High     | Partially Remediated
```

***

## Supplementary Deliverables

```
Attestation Report / Letter:
  Purpose: Provide evidence to the client's vendors or customers that a
           penetration test was conducted, without revealing sensitive details.
  Length:  1-2 pages maximum.
  Contents:
    - Date range of the assessment
    - Assessment type and scope description (general)
    - Number of findings by severity
    - General commentary on the environment's security posture
    - Tester/firm name and signature
  What to exclude:
    - Specific finding details
    - Compromised credentials or hashes
    - Internal hostnames or IP addresses
    - Any information that could assist a third-party attacker

Findings Spreadsheet:
  All finding fields from the report in tabular format.
  Client uses it for: sorting by severity, importing into ticketing systems,
  tracking remediation progress internally.
  Include: finding ID, title, severity, CVSS score, affected hosts, status.
  Exclude: executive summary, attack chain narratives, long descriptions.
  Add value: use pivot tables to show findings by severity category,
  by host, by finding type. Clients find this genuinely useful for prioritisation.

Slide Deck (Executive Presentation):
  Two versions may be needed:
    Technical version:   For the IT/security team, CISO
    Executive version:   For board, CEO, CFO, legal

  Executive presentation guidance:
    Numbers and graphs alone will lose the audience.
    Include one or two relevant anecdotes from real-world incidents in
    the same industry as your client.
    This makes the risk relatable without fear-mongering.
    Connect each key finding to a real business consequence.
    Keep it under 15 slides for a standard review meeting.

  Not appropriate for slide decks:
    Specific technical commands or payloads
    Full reproduction steps
    Raw scan output
    Anything that requires technical knowledge to interpret

Vulnerability Notification (out-of-band, during testing):
  When to issue one:
    Minimum trigger: any unauthenticated RCE on an internet-facing system.
    Also required for: sensitive data exposure, default/weak credentials
    on critical external services, anything with immediate significant business impact.
    Additional triggers should be agreed during the kickoff call.
    Set a baseline, communicate it to the client, let them adjust if needed.

  Contents (keep it brief):
    This is not the place for a full polished finding.
    Technical staff need to act immediately.
    Include:
      - One-paragraph description of what was found
      - The affected host and service
      - Evidence (screenshot or tool output)
      - CVSS score
      - Recommended immediate action
    The full finding writeup goes in the main report later.
    Purpose here is speed of communication, not completeness.
```

***

## Choosing the Right Report Depth

One of the more nuanced professional judgements in pentesting is calibrating the level of detail to the client's team.

```
Client maturity signals that guide report depth:

Mature security team (CISO, dedicated security engineers):
  - Wants full technical detail, exact commands, CVSS scores
  - Executive summary can be slightly more technical
  - Will ask detailed questions about specific findings
  - May challenge your severity ratings with solid reasoning

Less mature team (IT generalists, no dedicated security staff):
  - Needs more explanation of why each finding is dangerous
  - Reproduction steps must be exceptionally clear
  - Executive summary needs to be accessible to a non-technical CEO
  - More value in the near/medium/long-term recommendations section

Board-level engagement (executive briefing):
  - Focus entirely on business risk and financial/legal implications
  - Translate technical findings to operational impact
  - Three to five key messages maximum
  - Avoid all technical terminology without plain-language context

In every case:
  The report should be structured so any reader can get value
  from the section written for their level, without needing to read
  sections written for a different audience.
  This is the hallmark of a well-structured penetration test report.
```
