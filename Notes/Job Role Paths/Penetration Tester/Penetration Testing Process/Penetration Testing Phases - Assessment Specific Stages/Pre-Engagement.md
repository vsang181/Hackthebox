## Pre-Engagement

Pre-engagement is the most legally and operationally critical phase of any penetration test. Every document produced here is your legal authorisation to perform activities that would otherwise constitute criminal offences under the Computer Misuse Act 1990.  Getting this phase wrong exposes you to criminal liability, scope creep, unsatisfied clients, and unenforceable contracts. 

***

## Document Timeline

```
Initial Contact
      |
      v
1. NDA signed <- nothing confidential is discussed before this
      |
      v
2. Scoping Questionnaire sent to client
      |
      v
3. Pre-Engagement Meeting (discuss questionnaire responses)
      |
      v
4. Scoping Document produced
5. Penetration Testing Proposal / SoW produced
      |
      v
6. Rules of Engagement (RoE) finalised
7. Contractors Agreement (if physical testing is included)
      |
      v
Kick-Off Meeting
      |
      v
Testing begins
```

***

## NDA Types

| Type | Who Is Bound | When to Use |
|---|---|---|
| Unilateral | Only one party (usually tester) | Client wants to protect their data, no mutual obligation |
| Bilateral | Both parties | Standard for most engagements, protects both sides |
| Multilateral | Three or more parties | Consortium networks, multi-vendor environments |

A bilateral NDA is the most common and most appropriate for penetration testing engagements. It protects the tester's methodologies and tools as well as the client's data and findings.

***

## Who Has Authority to Commission a Test

This matters legally. If the person who hired you did not have authority, the authorisation is invalid. 
```
Authorised signatories typically include:
- Chief Executive Officer (CEO)
- Chief Technical Officer (CTO)
- Chief Information Security Officer (CISO)
- Chief Security Officer (CSO)
- Chief Risk Officer (CRO)
- Chief Information Officer (CIO)
- VP of Internal Audit / Audit Manager
- VP or Director of IT Security

Red flags: a request from a standard employee, a junior IT staff member,
or anyone without explicit budget and contractual authority

Always verify signatory authority before commencing any documentation.
If in doubt, ask for confirmation from the C-suite in writing.
```

***

## Scoping Questionnaire

The first document sent to the client, before any detailed discussion. Determines what they need, the size of the engagement, and allows accurate time and cost estimation. 

```
Assessment type selection (client checks all that apply):
[ ] Internal Vulnerability Assessment
[ ] External Vulnerability Assessment
[ ] Internal Penetration Test
[ ] External Penetration Test
[ ] Wireless Security Assessment
[ ] Web Application Security Assessment
[ ] Mobile Application Security Assessment
[ ] API Security Assessment
[ ] Physical Security Assessment
[ ] Social Engineering Assessment (phishing / vishing)
[ ] Red Team Assessment
[ ] Active Directory Security Assessment

Quantitative scoping questions:
- How many expected live hosts?
- How many IP addresses / CIDR ranges in scope?
- How many domains / subdomains in scope?
- How many wireless SSIDs in scope?
- How many web / mobile applications?
  - If authenticated testing: how many user roles?
- For phishing: how many target users?
  - Client-provided list or OSINT-gathered?
- For physical: how many locations? Geographically dispersed?
- For Red Team: what is the specific objective / flag?

Information level:
- Black box (minimal: IPs and domains only)
- Grey box (extended: URLs, hostnames, subnets, some credentials)
- White box (maximum: configs, source code, admin credentials)

Evasion level:
- Non-evasive (loud, fast, full speed ahead)
- Hybrid-evasive (start quiet, gradually increase noise to test detection)
- Fully evasive (ghost mode throughout)

Network access:
- Anonymous user on the network
- Standard domain user (credentials provided)
- Does NAC need to be bypassed?
```

***

## Pre-Engagement Meeting - Contract Checklist

This meeting translates the scoping questionnaire into a formal contract. Every item below must be discussed and documented. 

```
[ ] NDA - confirm signed before any discussion begins
[ ] Goals - major milestones, then granular sub-goals
[ ] Scope - domains, IP ranges, individual hosts, accounts, security systems
           - explicitly list out-of-scope systems too
[ ] Penetration Testing Type - explain options, make a recommendation, client decides
[ ] Methodologies - OSSTMM, OWASP, PTES, NIST, manual + automated
[ ] Testing Locations - remote via VPN or on-site
[ ] Time Estimation - start date, end date, time windows per phase
[ ] Third Parties - cloud providers, ISPs, hosting providers
                  - obtain and receive written confirmation of their permission
[ ] Evasive Testing - agreed evasion level, what detection testing is in scope
[ ] Risks - potential impact of testing, agreed limitations
[ ] Scope Limitations - systems critical to business continuity that must not be touched
[ ] Information Handling - HIPAA / PCI / GDPR / FISMA requirements
[ ] Contact Information - name, title, email, phone, office phone per person
                        - primary and secondary contacts per side
[ ] Lines of Communication - email, phone, video call, in-person
[ ] Reporting - format, audience, level of technical detail, presentation required?
[ ] Payment Terms - fees, milestones, invoicing schedule
```

***

## Rules of Engagement (RoE) - Full Checklist

The RoE is the operational document that governs everything during active testing. It is more granular than the contract and is the document you refer to during testing if questions arise about what you are allowed to do. 

```
[ ] Introduction - document purpose and overview
[ ] Contractor details - company name, lead tester name, job title
[ ] Penetration Tester names - full name of each tester on the engagement
[ ] Contact Information - all parties, mailing address, email, phone
[ ] Purpose - why this test is being conducted
[ ] Goals - specific objectives (access domain admin, reach file server, etc.)
[ ] Scope - every IP, CIDR, domain, URL explicitly listed
            include a separate "out of scope" section
[ ] Lines of Communication - agreed communication channels and frequency
[ ] Time Estimation - exact start and end dates/times
[ ] Time of Day to Test - business hours only, 24/7, or specific windows
[ ] Penetration Testing Type - External / Internal / VA / Social Engineering
[ ] Testing Locations - how connection to client network is established
[ ] Methodologies - OSSTMM, PTES, OWASP, etc.
[ ] Objectives / Flags - specific targets (user accounts, files, databases)
[ ] Evidence Handling - encryption requirements, secure transfer protocols
[ ] System Backups - confirm client has backups before testing starts
[ ] Information Handling - data classification requirements
[ ] Incident Handling - when to pause testing and call the emergency contact
[ ] Status Meetings - frequency, participants, format
[ ] Reporting - report type, target audience, technical depth
[ ] Retesting - dates for post-remediation retest if agreed
[ ] Disclaimers and Limitation of Liability - system damage, data loss clauses
[ ] Permission to Test - signed contract reference, contractor agreement
```

***

## Kick-Off Meeting

The final meeting before testing begins. Usually in person or on a scheduled video call with both technical and management stakeholders present. 

```
Attendees typically include:
Client side:
- CISO / IT Security lead (PoC)
- Technical support staff (sysadmins, network engineers, developers)
- Management representative

Tester side:
- Practice Lead / Project Manager
- Lead Penetration Tester(s)
- Account Executive (optional)

What gets covered:
1. Confirm scope has not changed since contract was signed
2. Confirm emergency contacts are available and reachable
3. Explain what logs/alerts will be generated during testing
4. Explain risk of accidental account lockouts during brute-force phases
5. Confirm client's expectation if a critical finding is discovered mid-test
6. Explain the pause/escalation process:

Pause triggers for External Tests:
- Unauthenticated RCE discovered
- SQL injection leading to sensitive data disclosure
- Any critical flaw before client can assess risk

Pause triggers for Internal Tests:
- A system becomes unresponsive due to testing
- Evidence of illegal content found on file shares
- Evidence of an active external threat actor in the network
- Evidence of a prior breach discovered during testing

7. Agree on status update frequency during the engagement
8. Confirm testing window times (start/end each day)
9. Client acknowledges testing will generate IDS/firewall alerts
```

***

## Contractors Agreement (Physical Assessments Only)

Physical testing introduces entirely different legal dimensions. Employees who do not know a test is happening will contact police if they catch someone attempting unauthorised physical access. This document is the equivalent of a "get out of jail free card" and must be carried on your person during any physical assessment. 

```
Physical assessment contractors agreement must include:
[ ] Introduction and purpose
[ ] Contractor and tester identities with photo ID reference
[ ] Physical addresses of all locations to be tested
[ ] Building names, floor numbers, room identifications
[ ] Physical components in scope (server rooms, reception, car parks)
[ ] Specific physical attack methods authorised (tailgating, badge cloning, lockpicking)
[ ] Timeline with precise dates and hours
[ ] Emergency contact who can verify authorisation to law enforcement
[ ] Notarisation (for high-risk or high-profile engagements)
[ ] Signed permission to test

Practical field requirement:
Always carry:
1. Physical copy of the contractors agreement
2. Client emergency contact's personal mobile number (not desk phone)
3. Client emergency contact's name so they can be reached 24/7

If detained by security or police:
- Stop all activity immediately
- Present the contractors agreement
- Ask client emergency contact to be called immediately
- Do not argue or resist
- Document the incident for inclusion in the final report
```

***

## Common Pre-Engagement Mistakes

```
Legal exposure risks:
- Sending a scoping questionnaire before NDA is signed
- Accepting verbal authorisation without written confirmation
- Not verifying that the person signing has actual authority
- Failing to obtain third-party cloud/hosting provider permission
- Allowing scope to be defined too vaguely ("test our network")

Operational risks:
- Not agreeing on out-of-scope systems (fragile production hosts)
- Not confirming client has backups before testing starts
- No agreed escalation process for critical findings
- No agreed process for system outages caused by testing
- Starting testing before all parties have countersigned the RoE

Reporting risks:
- Not clarifying report audience and technical depth at outset
- Not confirming retest scope and timeline in the contract
- No agreed data retention and destruction timeline post-engagement
```
