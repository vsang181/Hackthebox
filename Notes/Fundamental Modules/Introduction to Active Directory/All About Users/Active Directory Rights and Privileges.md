# Active Directory Rights and Privileges

In Active Directory, **rights** and **privileges** define what an account can *access* and what it can *do*. When these are misunderstood or poorly assigned, they frequently become the root cause of privilege escalation and full domain compromise.

Before you move further into enumeration or attacks, you need to be absolutely clear on the difference between the two and how Windows applies them.

---

## Rights vs Privileges

Although they are often used interchangeably, they are not the same thing.

### Rights

* Control **access to objects**
* Typically enforced through ACLs
* Examples:

  * Reading a file
  * Writing to a directory
  * Modifying a registry key

### Privileges

* Allow **actions to be performed**
* Granted directly or via group membership
* Examples:

  * Shutting down a system
  * Loading drivers
  * Debugging processes
  * Impersonating another account

Windows uses the term **User Rights Assignment**, but these are technically **privileges**, not access rights.

---

## Built-in Active Directory Groups

Active Directory includes many default security groups. Some of these grant extremely powerful capabilities and should be treated as **high-risk by default**.

Excessive or forgotten group membership is one of the most common weaknesses attackers abuse in real environments.

---

## Common Built-in Groups and Their Capabilities

| Group                              | Capabilities                                         |
| ---------------------------------- | ---------------------------------------------------- |
| Account Operators                  | Create and manage most user and group accounts       |
| Administrators                     | Full control of a computer or domain (on DCs)        |
| Backup Operators                   | Backup/restore any file, including SAM and NTDS.dit  |
| DnsAdmins                          | Manage DNS configuration                             |
| Domain Admins                      | Full control of the domain and all joined machines   |
| Domain Computers                   | All non-DC computer accounts                         |
| Domain Controllers                 | All DCs in the domain                                |
| Domain Guests                      | Built-in guest accounts                              |
| Domain Users                       | All user accounts                                    |
| Enterprise Admins                  | Forest-wide administrative control                   |
| Event Log Readers                  | Read system and security logs                        |
| Group Policy Creator Owners        | Create and modify GPOs                               |
| Hyper-V Administrators             | Full control of Hyper-V (dangerous with virtual DCs) |
| IIS_IUSRS                          | IIS worker process group                             |
| Pre-Windows 2000 Compatible Access | Legacy read access (often insecure)                  |
| Print Operators                    | Manage printers on DCs (historically abused)         |
| Protected Users                    | Extra protections against credential theft           |
| Read-only Domain Controllers       | RODC accounts                                        |
| Remote Desktop Users               | RDP access                                           |
| Remote Management Users            | WinRM access                                         |
| Schema Admins                      | Modify the AD schema (forest-wide impact)            |
| Server Operators                   | Manage services and shares on DCs                    |

---

## Example: Server Operators Group (Default State)

The **Server Operators** group is a good example of a group that looks harmless but is extremely dangerous if populated.

```powershell
Get-ADGroup -Identity "Server Operators" -Properties *
```

Key observations:

* Domain Local group
* Empty by default
* Members can manage services on Domain Controllers
* Often abused for privilege escalation if populated

---

## Example: Domain Admins Group

```powershell
Get-ADGroup -Identity "Domain Admins" -Properties * |
Select DistinguishedName, GroupCategory, GroupScope, Name, Members
```

Notes:

* Global security group
* Members automatically become local administrators on all domain-joined systems
* Primary target during almost every AD attack

---

## User Rights Assignment (Privileges)

Privileges can be assigned through:

* Group membership
* Local security policy
* Group Policy Objects (GPOs)

Some privileges are rarely needed and extremely dangerous if misused.

---

## Privileges Commonly Abused for Escalation

| Privilege                     | What It Enables             |
| ----------------------------- | --------------------------- |
| SeRemoteInteractiveLogonRight | Remote Desktop access       |
| SeBackupPrivilege             | Read protected system files |
| SeDebugPrivilege              | Dump LSASS memory           |
| SeImpersonatePrivilege        | Impersonate SYSTEM tokens   |
| SeLoadDriverPrivilege         | Load kernel drivers         |
| SeTakeOwnershipPrivilege      | Take ownership of objects   |

If you encounter any of these unexpectedly, assume escalation is possible.

---

## Viewing Assigned Privileges

After logging into a Windows host, you can list assigned privileges using:

```cmd
whoami /priv
```

What you see depends on:

* Group membership
* Whether the session is elevated
* User Account Control (UAC)

---

## Standard Domain User Example

```cmd
whoami /priv
```

Expected output:

* Very limited privileges
* No escalation-useful rights present

---

## Domain Admin (Non-Elevated)

In a non-elevated session, even Domain Admins appear restricted due to UAC.

This is intentional and prevents applications from running with full privileges by default.

---

## Domain Admin (Elevated)

Running an elevated shell reveals the true power of the account:

```cmd
whoami /priv
```

Notable privileges:

* SeDebugPrivilege
* SeImpersonatePrivilege
* SeCreateGlobalPrivilege

These are enough to fully compromise a system.

---

## Backup Operators Example

Members of **Backup Operators** may appear low-risk, but:

* They can log on locally to DCs
* They can create shadow copies
* They can extract credentials without touching LSASS

Even without elevation, this group should be treated as privileged.

---

## Key Takeaways

* Privileges matter more than passwords
* Built-in groups are frequent escalation paths
* Many environments forget to audit group membership
* Temporary access often becomes permanent risk

Best practice:

* Keep privileged groups empty by default
* Use separate admin and daily-use accounts
* Monitor changes to sensitive groups
* Assign privileges only when absolutely required
