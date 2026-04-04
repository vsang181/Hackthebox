## Penetration Testing Overview

A penetration test is an authorised, structured attempt to attack an organisation's IT systems using the same methods real attackers use, with the goal of uncovering vulnerabilities before malicious actors do. Unlike a vulnerability assessment which relies purely on automated scanning tools like Nessus or OpenVAS, a pentest combines automated and manual techniques tailored specifically to the target environment.

***

## Core Concepts

**The CIA Triad as the measuring stick** - every finding in a pentest report should be evaluated against its impact on:

```
Confidentiality  -> Can sensitive data be read by unauthorised parties?
Integrity        -> Can data be modified without authorisation?
Availability     -> Can systems/services be disrupted or taken offline?
```

**Pentest vs Vulnerability Assessment vs Red Team**

| Assessment Type | Approach | Goal |
|---|---|---|
| Vulnerability Assessment | Automated scanning only | Identify known vulnerabilities |
| Penetration Test | Manual + automated, all vulns | Find and validate all weaknesses |
| Red Team | Scenario-driven, stealthy | Test detection and response capability |
| Purple Team | Collaborative with defenders | Improve both attack and defence together |

***

## Risk Management Context

Pentesting sits inside an organisation's broader risk management programme. The four responses to risk are:

```
Accept   -> Risk acknowledged, cost of fixing exceeds impact (residual/inherent risk)
Transfer -> Cyber insurance, third-party liability contracts
Avoid    -> Remove the system or service entirely
Mitigate -> Apply controls to reduce likelihood or impact (patching, segmentation, etc.)

Inherent risk = risk that remains even after all reasonable controls are in place
Our job as testers = identify where that inherent risk is higher than the client assumes
```

***

## Legal and Ethical Boundaries

This is non-negotiable. A pentest without written authorisation is a criminal offence under the Computer Misuse Act 1990 (UK) and equivalent laws in other jurisdictions.

```
Before any testing starts, you must have:
1. Signed Statement of Work (SoW) or Rules of Engagement (RoE)
2. Written scope defining exactly which systems are in scope
3. Emergency contacts for the client if something goes wrong
4. Confirmation of asset ownership (client must own what they ask you to test)
5. Third-party approval if client uses hosted infrastructure:
   - AWS, Azure, GCP each have their own penetration testing policies
   - AWS no longer requires prior approval for most testing within your own account
   - Always verify per provider - policies change

Never test systems not explicitly listed in scope.
If you stumble onto a third-party system, stop and notify the client immediately.
```

***

## Testing Perspectives

```
External Pentest:
- Attacker perspective: anonymous user on the internet
- Goal: breach the perimeter, access internal network or sensitive data
- Common starting point: from your own host via VPN or a VPS
- Stealth level varies by client - some want noise, some want ghost mode

Internal Pentest:
- Perspective: attacker already inside the network
- May follow a successful external pentest
- Or may start from an assumed breach scenario (given low-priv domain user creds)
- May require physical presence if systems have no internet access
```

***

## Information Levels (Black/Grey/White Box)

```
Blackbox:
- Given: IP ranges, domain names only
- Approach: full recon from scratch (DNS, port scanning, OSINT)
- Time required: longest
- Mirrors: real external attacker with no inside knowledge

Greybox:
- Given: IP ranges + some URLs, hostnames, subnets, user accounts
- Approach: targeted testing with partial context
- Most common real-world engagement type

Whitebox:
- Given: everything - source code, configs, admin credentials, network diagrams
- Approach: deep-dive code review + authenticated testing
- Time required: most thorough, finds most vulnerabilities
- Mirrors: insider threat or post-compromise attacker with full access

Red Team:
- Scenario-driven (e.g., "can you access the CFO's laptop?")
- May include phishing, physical access, social engineering
- Tests detection and incident response, not just technical defences
- Often time-boxed over weeks or months
```

***

## Testing Environment Categories

```
Network          -> External/internal infrastructure, firewall rules, segmentation
Web Application  -> OWASP Top 10, auth, injection, session management
Mobile           -> iOS/Android app analysis, API calls, local storage
API              -> REST/SOAP/GraphQL endpoints, authentication tokens
Thick Clients    -> Desktop applications, memory analysis, traffic interception
IoT              -> Embedded devices, firmware, default credentials
Cloud            -> AWS/Azure/GCP misconfigurations, IAM permissions
Source Code      -> Static analysis, hardcoded credentials, insecure functions
Physical         -> Tailgating, lock picking, badge cloning
Employees        -> Phishing, vishing, pretexting
Hosts/Servers    -> OS hardening, local privilege escalation (this module)
Security Policies -> Gap analysis, documentation review
Firewalls/IDS/IPS -> Rule bypass testing, evasion techniques
```

***

## The Penetration Tester's Role

```
You ARE:
- A trusted advisor documenting vulnerabilities with reproduction steps
- A reporter of findings with risk ratings and remediation recommendations
- Responsible for protecting any sensitive data encountered (PII, card data, etc.)

You are NOT:
- The person who applies patches or fixes code
- A monitoring solution (pentest is a point-in-time snapshot)
- Authorised to act outside the agreed scope under any circumstances

If you find PII (names, addresses, salaries, card numbers):
- Document existence and location only
- Do not exfiltrate unnecessarily
- Recommend immediate remediation in your report
- Note in the report that data encryption and password rotation are required
- UK Data Protection Act 2018 / GDPR applies - handle accordingly
```

***

## Pentest Report Fundamentals

```
A pentest report must include at minimum:
1. Executive Summary     -> Non-technical, risk-focused, for C-level readers
2. Scope and Methodology -> What was tested, how, from which perspective
3. Findings              -> Each vulnerability with:
   - Title and risk rating (Critical/High/Medium/Low/Informational)
   - Description
   - Evidence (screenshots, command output)
   - Step-by-step reproduction
   - Remediation recommendation
4. Risk ratings          -> Aligned to CVSS or your firm's own matrix
5. Point-in-time statement -> "This assessment reflects the security posture
                               as of [date] and does not constitute ongoing monitoring"
6. Appendices            -> Raw tool output, supporting evidence
```
