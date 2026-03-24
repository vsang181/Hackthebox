## Access Control List (ACL) Abuse Primer

ACLs are lists that define who has access to which AD object and at what level. The individual settings within an ACL are called Access Control Entries (ACEs), and each ACE maps to a security principal (user, group, or process) and defines the rights granted to that principal. Every object in AD has an ACL, and most have multiple ACEs because multiple principals can interact with the same object.

***

## Two Types of ACLs

- **Discretionary Access Control List (DACL)**: defines which security principals are granted or denied access to an object. If no DACL exists, all users get full rights. If a DACL exists but has no ACE entries, all access is denied.
- **System Access Control List (SACL)**: used by administrators to log access attempts made to secured objects, visible under the Auditing tab in Active Directory Users and Computers.

***

## Access Control Entries (ACEs)

There are three main ACE types applied to securable objects in AD:

| ACE Type | Description |
|----------|-------------|
| Access denied ACE | Used in a DACL to explicitly deny a user or group access to an object |
| Access allowed ACE | Used in a DACL to explicitly grant a user or group access to an object |
| System audit ACE | Used in a SACL to generate audit logs when a user or group attempts to access an object |

Each ACE is made up of four components:

- The SID of the user or group (the security principal)
- A flag indicating the ACE type (denied, allowed, or audit)
- Inheritance flags specifying whether child containers or objects inherit the ACE from the parent
- A 32-bit [access mask](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-dtyp/7a53f60e-e730-4dfe-bbe9-b21b62eb790b) defining the specific rights granted

ACLs are evaluated top to bottom, and the first access denied entry encountered stops further evaluation.

***

## Why ACEs Matter for Attackers

ACL misconfigurations are particularly dangerous because they cannot be detected by standard vulnerability scanners, they often persist untouched for years, and they are frequently unknown even to the organisations that created them. Installing software like Microsoft Exchange automatically introduces ACL changes into the domain, many of which grant excessive rights to broad groups.

The abusable ACEs covered in this module and their corresponding [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) methods are:

| ACE / Right | PowerView Abuse Method | What It Allows |
|-------------|----------------------|----------------|
| `ForceChangePassword` | `Set-DomainUserPassword` | Reset a user's password without knowing the current one |
| `GenericWrite` | `Set-DomainObject` | Write to any non-protected attribute; assign an SPN for Kerberoasting, or add members to a group |
| `AddSelf` | `Add-DomainGroupMember` | Add yourself to a security group |
| `GenericAll` | `Set-DomainUserPassword` or `Add-DomainGroupMember` | Full control over the target object; modify group membership, force a password change, or perform targeted Kerberoasting |
| `WriteOwner` | `Set-DomainObjectOwner` | Change ownership of an object |
| `WriteDACL` | `Add-DomainObjectACL` | Modify the DACL of an object, granting yourself or others additional rights |
| `AllExtendedRights` | `Set-DomainUserPassword` or `Add-DomainGroupMember` | Includes all extended rights such as password resets and group membership changes |

`GenericAll` over a computer object where [LAPS](https://www.microsoft.com/en-us/download/details.aspx?id=46899) is deployed means you can read the LAPS-managed local administrator password and gain local admin access to that machine.

***

## ACL Attacks in Practice

ACL abuse serves three main purposes during an assessment: lateral movement, privilege escalation, and persistence. The three most common attack scenarios are:

- **Abusing password reset permissions**: Help Desk and IT accounts are routinely granted password reset rights. Taking over one of these accounts may allow you to reset the password for a more privileged account and take over the domain.
- **Abusing group membership management**: Many IT accounts have rights to add or remove users from specific groups. Adding an account you control into a privileged built-in AD group like Backup Operators or Domain Admins is a fast path to full domain compromise.
- **Excessive user rights**: Legacy configurations, software installs, or convenience-driven changes often leave user, computer, or group objects with unintended elevated rights. These go unreviewed and are frequently the most impactful findings on a mature engagement.

***

## Enumerating ACLs

[BloodHound](https://github.com/BloodHoundAD/BloodHound) is the most efficient tool for visualising ACL attack paths. It maps edges such as `ForceChangePassword`, `GenericAll`, `GenericWrite`, `WriteDACL`, `WriteOwner`, `AddSelf`, and others, and shows the shortest path between a compromised account and Domain Admin. PowerView can enumerate the same relationships at a more granular, targeted level.

Beyond the common edges, you may encounter less familiar rights such as [ReadGMSAPassword](https://bloodhound.specterops.io/resources/edges/read-gmsa-password) (abusable with [GMSAPasswordReader](https://github.com/rvazarkar/GMSAPasswordReader)), [Unexpire-Password](https://learn.microsoft.com/en-us/windows/win32/adschema/r-unexpire-password), or [Reanimate-Tombstones](https://learn.microsoft.com/en-us/windows/win32/adschema/r-reanimate-tombstones). Reviewing the full [BloodHound edges reference](https://bloodhound.specterops.io/resources/edges/overview) and [Active Directory Extended Rights](https://learn.microsoft.com/en-us/windows/win32/adschema/extended-rights) documentation helps when you encounter something unusual.
