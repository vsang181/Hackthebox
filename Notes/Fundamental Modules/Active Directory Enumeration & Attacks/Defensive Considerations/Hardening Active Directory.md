# Hardening Active Directory

Effective AD hardening is less about purchasing security products and more about building a solid baseline: proper documentation, defined processes, and consistent technology controls. Without these fundamentals in place, even the most advanced SIEM or EDR tool provides limited value. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Step One: Document and Audit

An AD environment you do not fully understand cannot be properly defended. The following should be audited at minimum annually, ideally every quarter: [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

- Naming conventions for OUs, computers, users, and groups
- DNS, network, and DHCP configurations
- All GPOs and the objects they apply to
- FSMO role assignments
- Full application inventory
- All enterprise hosts and their physical/logical location
- Trust relationships with other domains or external entities
- All users with elevated permissions

***

## People

Users consistently represent the weakest link regardless of technical controls in place. The goal here is reducing the attack surface created by human behaviour and over-privileged accounts. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

- Enforce a strong password policy with filters blocking common words, seasonal terms, and the company name; deploy an enterprise password manager where feasible
- Rotate service account passwords periodically
- Disable local admin access on workstations unless a documented business need exists
- Disable the default RID-500 local admin account and create a new admin account managed by LAPS
- Implement tiered administration: admins should not use the same account for both privileged tasks and daily work
- Audit and trim privileged groups aggressively; an organisation with 50+ Domain Admins has a significant exposure problem
- Place high-value accounts in the `Protected Users` group
- Disable Kerberos delegation for administrative accounts

### Protected Users Group

The `Protected Users` group, available since Windows Server 2012 R2, enforces a set of non-configurable protections on its members regardless of other policy settings: [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

- Members cannot be configured for constrained or unconstrained delegation
- CredSSP will not cache plaintext credentials in memory
- Windows Digest will not cache plaintext passwords
- NTLM authentication is blocked for members; DES and RC4 Kerberos keys are also blocked
- TGT long-term keys and plaintext credentials are not cached after acquisition
- TGT renewal is capped at 4 hours, preventing long-lived ticket abuse

Check current membership:

```powershell
Get-ADGroup -Identity "Protected Users" -Properties Name,Description,Members
```

One important caveat: placing accounts in `Protected Users` without staged testing can cause authentication failures and account lockouts. NTLM-dependent applications will break for these users, so this should be rolled out carefully with application compatibility verified first. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Processes

Technical controls mean little without defined policies and procedures that govern how the environment is operated and maintained. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Key process areas to address:

- AD asset management: host audits, asset tagging, periodic inventory to prevent ghost machines
- Access control policies covering account provisioning and de-provisioning, with multi-factor authentication requirements
- Standardised host build and decommission procedures using hardened gold images
- AD cleanup policy: define whether former employee accounts are disabled or deleted, and set a timeline for removing stale records
- Decommissioning procedures for legacy systems, particularly domain-integrated applications like on-premises Exchange being migrated to O365
- A recurring schedule for auditing users, groups, and hosts

***

## Technology Controls

| Control | Attacks Mitigated |
|---------|------------------|
| Use gMSA/MSA instead of standard service accounts | Kerberoasting |
| Set `ms-DS-MachineAccountQuota` to 0 | NoPac, RBCD attacks |
| Disable Print Spooler where not needed | PrintNightmare, Printer Bug/MS-RPRN |
| Disable unconstrained delegation | TGT harvesting, cross-forest attacks |
| Enable SMB signing and LDAP signing | NTLM relay, PetitPotam, LDAP relay |
| Disable NTLM on DCs where possible | Pass-the-hash, NTLM relay chains |
| Enable Extended Protection for Authentication on AD CS | PetitPotam, ESC8 |
| Set `RestrictNullSessAccess = 1` | Anonymous/null session enumeration |
| Restrict access to DCs via hardened jump hosts | Direct lateral movement to DCs |
| Audit SYSVOL scripts and AD description fields | Cleartext credential harvesting |
| Run BloodHound, PingCastle, Grouper periodically | Misconfiguration identification |
| Quarterly penetration tests (annually at minimum) | All attack categories |

***

## Protections Mapped to Attack Techniques

| TTP | MITRE Tag | Key Defences |
|-----|----------|-------------|
| External Reconnaissance | T1589 | Scrub document metadata, sanitise job postings, control public DNS/BGP data |
| Internal Reconnaissance | T1595 | NIDS/firewall alerting on scan patterns, disable ICMP responses, SIEM correlation |
| Poisoning/Relay | T1557 | SMB signing, LDAP signing, strong encryption, disable NTLM |
| Password Spraying | T1110/003 | Monitor Event IDs 4624/4648, account lockout policy, MFA, strong password policy |
| Credentialed Enumeration | TA0006 | Monitor for anomalous CLI use, RDP hopping, file movement; network segmentation |
| Living off the Land | N/A | Network traffic baseline, AppLocker policy, behavioural anomaly detection |
| Kerberoasting | T1558/003 | Use AES over RC4, strong service account passwords, gMSA accounts, audit group memberships |
| ASREPRoasting | T1558/004 | Audit `DONT_REQ_PREAUTH` flag, enforce pre-authentication, strong passwords |

***

## Reading the MITRE ATT&CK Framework

The framework uses a two-tier reference system that is worth understanding for both offensive planning and client reporting: [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

- `TA####` is a **Tactic**, the overarching goal (e.g., `TA0006` = Credential Access)
- `T####` is a **Technique** under a tactic (e.g., `T1558` = Steal or Forge Kerberos Tickets)
- `T####.00#` is a **Sub-technique** (e.g., `T1558.003` = Kerberoasting specifically)

So Kerberoasting maps to `TA0006/T1558.003`. Each entry in the framework includes real-world usage examples, detection guidance, and mitigation recommendations, making it a practical reference for both attackers planning engagements and defenders building detection logic. Each finding in a penetration test report that includes a MITRE tag gives defenders a direct path to Microsoft and vendor guidance on how to respond to that specific technique. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)
