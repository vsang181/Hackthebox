# Introduction to Active Directory Enumeration & Attacks

Apologies Victor, here is the rewrite:

Active Directory has been the dominant enterprise identity and access management platform since its introduction with Windows Server 2000, and despite being over two decades old, it remains deeply embedded in the vast majority of corporate Windows environments worldwide. Understanding how to enumerate and attack it is not a niche specialisation. It is a core competency for any penetration tester working in enterprise environments.

## What AD Is and Why It Matters

Active Directory is built on the X.500 directory standard and LDAP, implementing a distributed, hierarchical structure for managing users, computers, groups, policies, and trusts across an organization. Every authentication request, every access control decision, and every policy enforcement in a Windows domain runs through it. Because AD is designed to make resources easy to find and access, it is inherently difficult to lock down without breaking legitimate functionality. Microsoft has reported over 2000 vulnerabilities tied to CVEs across its product suite in recent years, and AD's complexity means misconfigurations in services, permissions, and default settings are persistently common across organizations of every size.

The broader IAM market continues to grow, but AD's integration into both on-premises infrastructure and hybrid cloud environments through Azure AD Connect means it is not going away. An attacker who can compromise AD effectively owns the identity fabric of the entire organization.

## Three Real-World Attack Chains

The scenarios described in this module introduction are worth examining closely because they illustrate how iterative, patient enumeration leads to full domain compromise without relying on any zero-day exploit. Each one follows a logical chain from limited access to Domain Admin.

### Scenario 1 - Kerberoasting to Credential Relay

Starting from a single SYSTEM-level foothold on a domain-joined host, the path went through:

1. Standard domain enumeration reveals Service Principal Names (SPNs) registered on service accounts
2. A Kerberoasting attack requests TGS tickets for those accounts, requiring no special privileges since any authenticated domain user can request service tickets
3. Offline cracking with Hashcat using the d3ad0ne ruleset cracks one ticket overnight, yielding a plaintext password for a low-privilege account
4. That account has write access to file shares, where SCF files are dropped to capture NetNTLMv2 hashes from anyone browsing them
5. Responder captures a hash from a user that BloodHound reveals is a Domain Admin

The entire chain ran through nothing but misconfigurations: a service account with a crackable password, overly permissive file share access, and a Domain Admin browsing file shares in a way that exposed their hash.

### Scenario 2 - Password Spraying to Pass-the-Ticket

This chain demonstrates why enumerating the password policy before spraying is non-negotiable:

1. An SMB null session via `enum4linux` returns the full domain user list and password policy
2. Knowing the policy (8 character minimum, complexity required) prevents lockouts and narrows the wordlist to plausible passwords
3. `Spring@18` produces a hit on a low-privilege account
4. BloodHound identifies several hosts where this account has local admin access and shows a Domain Admin has an active session on one
5. Rubeus extracts the Kerberos TGT for the Domain Admin from that host's memory
6. A pass-the-ticket attack authenticates as the Domain Admin
7. Nested group membership in a trusting domain means the same credentials grant administrative access there too

### Scenario 3 - Username Enumeration to Shadow Credentials to DCSync

This is the most technically sophisticated chain of the three:

1. With no initial foothold, `linkedin2username` generates a candidate username list from the company's LinkedIn presence, combined with statistically-likely-usernames lists
2. `Kerbrute` validates 516 real domain accounts without triggering lockouts, since it uses Kerberos pre-authentication requests rather than LDAP bind attempts
3. A single password spray hit on `Welcome2021` gets a foothold
4. BloodHound shows all domain users have RDP access to one host, an immediate escalation in access
5. `DomainPasswordSpray` from inside the domain sprays a broader user list and returns several hits with `Fall2021`
6. One of those accounts is in the Help Desk group, which has `GenericAll` rights over the Enterprise Key Admins group
7. Enterprise Key Admins has `GenericAll` over a Domain Controller
8. Adding the controlled account to Enterprise Key Admins inherits those DC rights
9. The Shadow Credentials attack abuses Key Trust account mapping to retrieve the NT hash for the DC machine account
10. With the DC machine account hash, DCSync replicates all NTLM password hashes from the domain

## What This Module Covers

The attack chains above involve many techniques, some within this module and some beyond its scope. The focus here is building the foundational skill set required to execute the early and mid stages of these chains:

- Manual and tool-assisted AD enumeration using native Windows tools, WMI, DNS, Sysinternals, PowerView, and BloodHound
- Password attacks including spraying and Kerberoasting, with proper operational care around lockout policies
- Credential capture and relay with Responder
- Username enumeration with Kerbrute
- Understanding ACL-based attack paths as surfaced by BloodHound

Equally important is the requirement to operate from both Windows and Linux, and to fall back on native tooling when custom tools are blocked. Many real engagements involve working from a managed client workstation where you cannot run your own binaries. Knowing how to enumerate AD with nothing but built-in Windows commands is what separates a capable tester from one who is helpless when their toolkit is blocked.
