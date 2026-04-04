## How to Write Up a Finding

A finding is the core unit of value in a penetration test report. Every hour of technical work during an engagement ultimately exists to produce findings that a client can act on. If a finding is poorly written, the work behind it is wasted. 
***

## Required Elements Per Finding

Every finding must contain these fields, customised to the specific client environment. A generic finding copied from a database without modification misrepresents the actual risk. 

```
Mandatory fields:

1. Title
   Specific, professional, describes the exact issue.
   "Weak Kerberos Authentication (Kerberoasting)" - good
   "Kerberos issue found" - bad
   "AD problem" - useless

2. Severity Rating
   Critical / High / Medium / Low / Informational
   Justify the rating in the finding text.
   Account for the specific client environment, not just CVSS alone.
   A medium CVSS finding on an internet-facing PCI system may be High in context.

3. Description
   What is the vulnerability?
   Why does it exist? (root cause, not just the symptom)
   What platform, service, or application is affected?
   Written for a technical reader who may not be a penetration tester.
   Educate them. Many will never have heard of Kerberoasting or LLMNR spoofing.

4. Impact
   What can an attacker do if they exploit this?
   What data, systems, or business processes are at risk?
   Be specific about the consequences, not vague.
   "Could lead to unauthorised access" is useless.
   "Allows any authenticated domain user to retrieve a TGS ticket for
    service accounts and crack it offline, potentially revealing cleartext
    credentials for accounts with elevated privileges" is actionable.

5. Affected Systems
   Hostname, IP address, service, port, URL.
   List every affected host.
   If a finding affects many hosts, list them in an appendix
   and reference it from the finding.

6. Remediation
   Address the root cause, not the symptom.
   Be specific and actionable.
   Include multiple options where possible (free and paid).
   Include a warning if the fix carries its own risk (e.g., registry changes,
   GPO modifications that should be tested before wide deployment).

7. References
   One or more external links for further reading.
   See criteria in the references section below.

8. Reproduction Steps with Evidence
   Numbered steps exact enough to follow without guidance.
   Screenshots and command output inline.
   Written with the assumption that the reader has never seen the tool before.

Optional but valuable:
   CVE number (where applicable)
   CWE classification (class of vulnerability)
   MITRE ATT&CK technique ID
   CVSS base score with breakdown
   Ease of exploitation / probability of attack in this environment
```

***

## Reproduction Steps - The Most Commonly Underdeveloped Section

The reproduction steps are what allow the remediation team to confirm the fix works. If they cannot reproduce the finding independently, they cannot verify their fix. 

```
Principles for writing reproduction steps:

1. Break every step into its own figure
   One command or action per screenshot or code block.
   Multiple steps in one figure confuses a reader who does not know the tools.
   The reader needs to understand exactly what happened at each point.

2. Capture full tool setup before execution
   If a Metasploit module is used: screenshot the full module configuration
   before the run command. The reader needs to see what the settings
   were, not just what the output was.

3. Write a narrative between each figure
   Do not try to explain what happened in the figure caption.
   Do not stack consecutive figures without explanation.
   After each figure: one to three sentences explaining what just happened
   and what it means in the context of the assessment.
   "The above output shows the NTLMv2 hash for bsmith was successfully
    captured. This hash can now be taken offline for cracking."

4. Do not assume the reader knows what output means
   They may not know what NTLMv2-SSP Hash means.
   They may not know that a $krb5tgs$ prefix identifies a TGS ticket.
   Explain it. A brief sentence is enough.

5. Offer alternative tools
   Mention alternative tools that produce the same result.
   Do not perform the full exploit twice in the report.
   Just name the tool and provide a reference link.
   This is useful for clients who may be working from Windows
   and prefer a PowerShell equivalent to a Linux tool.

6. Evidence must be completely defensible
   Every screenshot must prove its own context:
   - What system (hostname in prompt or via hostname / ipconfig output)
   - What user (whoami or prompt showing username)
   - What URL or application (address bar visible in browser screenshots)
   - What was demonstrated (the action and its result both visible)

   Insufficient: screenshot of a basic auth login prompt
   Sufficient:   screenshot of the login prompt with test credentials,
                 plus Wireshark capture showing cleartext credentials in the packet

   Insufficient: screenshot of an admin panel with no URL bar
   Sufficient:   same screenshot with URL bar showing the target IP/domain

7. Client must be able to copy/paste key inputs
   If the finding involves a web payload or specific command, the client
   needs to copy it exactly to reproduce the finding.
   A screenshot of a payload in Burp Suite cannot be copy-pasted.
   Present the payload as text in the report, not only as a screenshot.

8. Browser hygiene for web application screenshots
   Turn off your bookmarks bar.
   Disable personal or unprofessional browser extensions.
   Use a dedicated testing browser profile.
   Personal bookmarks and extensions visible in screenshots look unprofessional.
```

***

## Effective Remediation Recommendations

The remediation section is where the finding converts from "here is a problem" to "here is how to fix it." Vague remediation advice wastes the client's time and undermines your credibility. 

```
The quality spectrum:

Bad:    "Reconfigure your registry settings to harden against this."
Better: "Disable LLMNR via Group Policy."
Good:   "To remediate this finding:
         1. Open Group Policy Management Console
         2. Edit the Default Domain Policy or create a new GPO linked at the domain level
         3. Navigate to Computer Configuration > Policies >
            Administrative Templates > Network > DNS Client
         4. Enable 'Turn off multicast name resolution'
         For NBT-NS, disable via Network Adapter settings or:
         HKLM\SYSTEM\CurrentControlSet\Services\NetBT\Parameters\Interfaces\<ID>
         Set NodeType to 0x2 (P-Node)
         Note: Test these changes in a pilot group before domain-wide deployment.
         Reference: https://docs.microsoft.com/en-us/troubleshoot/windows-server/..."

Why the detail matters:
  Your client's IT team may not have penetration testing experience.
  Making them spend hours figuring out what you meant by "reconfigure registry"
  frustrates them and reflects poorly on you.
  Specific remediation advice shows you did your homework.
  It builds confidence, increases repeat business, and makes you more comfortable
  answering questions during the report review meeting.

Include a risk warning when appropriate:
  Any finding whose remediation involves:
  - Registry edits
  - GPO changes
  - Service restarts
  - Firewall rule modifications
  Should include: "These changes should be tested in a non-production environment
  or small pilot group before broad deployment to avoid unintended disruption."
  This shows the client you have their operational continuity in mind.

Handle budget constraints honestly:
  Bad:   "Implement [expensive commercial tool] to fix this."
  Good:  "Several approaches exist. The affected software vendor has published
          a free workaround (link below). Commercial PAM tools such as those
          offered by several vendors can automate enforcement at scale, but
          these may be cost-prohibitive. The vendor workaround provides
          adequate protection as an interim measure while a longer-term
          solution is evaluated."

  Never leave a client with only a paid option when a free path exists.
  If you only name expensive solutions, clients without budget
  will simply leave the finding open.

Tie remediation to root cause:
  Symptom:   Domain Admin account using Password123
  Bad fix:   "Change this account's password."
  Root cause fix: "Implement a Group Policy password policy requiring a
                   minimum of 15 characters with complexity requirements.
                   Additionally, implement a privileged access workstation
                   (PAW) programme for all Domain Admin accounts and enforce
                   MFA for all privileged access."
```

***

## Selecting Quality References

Every finding should include at least one external reference. The quality of the reference reflects on the quality of the report. 
```
Good reference criteria:

  Vendor-agnostic where possible:
    Do not link to a vendor's website for a generic issue unless it is
    the authoritative source (e.g., Microsoft for Windows GPO guidance).
    Vendor-written articles often focus on selling their product rather
    than explaining the issue.

  Comprehensive and actionable:
    The linked article should explain what the finding is, why it is a problem,
    and how to fix it.
    Do not link to articles behind a paywall.
    Do not link to pages that give partial information without paying.

  Direct and concise:
    The reader has a problem to solve. They do not want to read through
    15 paragraphs before reaching the relevant content.
    Avoid links to entire standards documents (full NIST 800-53, full RFC).
    Link to the specific section if possible.

  Clean and trustworthy source:
    OWASP, NIST, Microsoft Docs, vendor advisories, CIS Benchmarks,
    established security blogs from reputable firms.
    Avoid sites with aggressive ads, pop-ups, or questionable content.
    Your client will click these links. What they see reflects on you.

  Consider writing your own source material:
    If you blog about findings you commonly discover, you can reference
    your own well-researched articles.
    This builds your professional reputation and keeps clients
    off competitors' websites.

Strong reference sources by finding type:
  Active Directory / Windows:
    https://docs.microsoft.com/en-us/security/
    https://www.adsecurity.org/
    https://attack.mitre.org/
    https://www.cisecurity.org/benchmark/microsoft_windows_server

  Web application:
    https://owasp.org/www-project-top-ten/
    https://cheatsheetseries.owasp.org/
    https://portswigger.net/web-security

  CVE-specific:
    https://nvd.nist.gov/vuln/detail/CVE-XXXX-XXXXX
    https://www.cvedetails.com/
    Vendor security advisory pages

  General:
    https://www.cisa.gov/known-exploited-vulnerabilities-catalog
    https://www.sans.org/reading-room/
```

***

## Anatomy of a Well-Written Finding

Using the Kerberoasting finding from the module as a model: 

```
Title:          Weak Kerberos Authentication (Kerberoasting)
Severity:       High

Description:
  Any authenticated domain user can request a Kerberos TGS ticket for any
  service account configured with a Service Principal Name (SPN). The ticket
  is encrypted with the service account's NTLM password hash. An attacker
  can request these tickets without generating alerts and crack them offline
  using tools like Hashcat. If the service account uses a weak password,
  the cleartext password is recovered without any interaction with the target.
  Service accounts typically hold elevated privileges, making them high-value targets.

Impact:
  Successful exploitation allows an attacker to retrieve cleartext credentials
  for service accounts that may hold local administrator rights over servers,
  access to databases containing sensitive data, or other elevated domain privileges.
  During this assessment, the mssqlsvc service account was cracked and used to
  gain local administrator access on SQL01, which led to the recovery of additional
  credentials and ultimately domain compromise.

Affected Systems:
  INLANEFREIGHT.LOCAL domain (3 Kerberoastable service accounts identified:
  mssqlsvc, sqlprod, backupjob)

Remediation:
  1. Audit all accounts configured with SPNs:
     Run: Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
  2. Remove SPNs from accounts that do not require them
  3. For required service accounts, implement passwords of 25+ random characters
     managed by a privileged access management (PAM) solution
  4. Where possible, replace traditional service accounts with Group Managed
     Service Accounts (gMSA), which automatically rotate to strong 120-character
     passwords - Microsoft documentation linked below
  5. Monitor for TGS requests to service accounts (Event ID 4769)
     and alert on unusual volumes

References:
  https://attack.mitre.org/techniques/T1558/003/
  https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/
  https://www.adsecurity.org/?p=2011

Reproduction Steps:

Step 1: Enumerate Kerberoastable accounts
  Using credentials for bsmith (standard domain user), the tester
  enumerated accounts with SPNs using GetUserSPNs.py:

  GetUserSPNs.py INLANEFREIGHT.LOCAL/bsmith -dc-ip 192.168.195.204

  [screenshot showing SPN enumeration output with mssqlsvc, sqlprod, backupjob]

  The above output identifies three service accounts with SPNs configured.
  These accounts are vulnerable to a Kerberoasting attack.

Step 2: Request the TGS ticket for mssqlsvc
  The tester performed a targeted Kerberoasting attack to retrieve
  the Kerberos TGS ticket for the mssqlsvc account:

  GetUserSPNs.py INLANEFREIGHT.LOCAL/bsmith -dc-ip 192.168.195.204 -request-user mssqlsvc

  [screenshot showing $krb5tgs$ hash output]

  The hash beginning with $krb5tgs$23$ is an RC4-encrypted TGS ticket
  that can be cracked offline.

Step 3: Crack the hash offline
  The tester used Hashcat to attempt to crack the hash using the
  rockyou.txt wordlist:

  hashcat -m 13100 mssqlsvc_tgs /usr/share/wordlists/rockyou.txt

  [screenshot showing Hashcat output with <REDACTED> as cracked password]

  The password was successfully cracked, granting access to the mssqlsvc account.

Alternative tools: Rubeus (Windows), CrackMapExec (enumeration phase)
```

***

## Common Finding Quality Failures

```
Failure 1: Vague impact statement
  Bad:   "This could allow an attacker to gain access to the system."
  Good:  "An unauthenticated attacker on the internal network can exploit
          this to achieve remote code execution as SYSTEM on WEB01, from
          which they can pivot to additional internal hosts."

Failure 2: Generic remediation that does not match the finding
  Bad:   "Update software and harden configurations."
  Good:  Specific registry paths, GPO settings, or configuration steps
         with warnings about testing before production deployment.

Failure 3: Evidence that does not prove the claim
  Bad:   Screenshot of an admin login page = proves basic auth is configured
  Good:  Screenshot of login page + Wireshark capture showing plaintext
         credentials in the HTTP request = proves transmission in clear text

Failure 4: No hostname or user context in screenshots
  Any screenshot without hostname, IP, and current user context
  can be challenged as not originating from the client environment.

Failure 5: Finding adapted from a template but not customised
  Bad:   "The default credentials admin:admin were found on a web application."
  Good:  "The Tomcat Manager application at http://10.10.10.15:8080/manager/html
          was found to accept the default credentials tomcat:tomcat, allowing
          an authenticated attacker to deploy a WAR file and achieve remote
          code execution as the service account running Tomcat on WEBDEV01."

Failure 6: Recommending a specific commercial product
  Bad:   "Deploy CrowdStrike Falcon to detect and prevent this attack."
  Good:  "Deploy an endpoint detection and response (EDR) solution capable
          of monitoring for anomalous Kerberos ticket requests (Event ID 4769).
          Several commercial and open-source solutions provide this capability."

Failure 7: Blank or placeholder CVSS score
  If your report template includes a CVSS field, fill it in every time.
  A blank CVSS field in a polished report looks like careless copy-paste
  from a template.
```
