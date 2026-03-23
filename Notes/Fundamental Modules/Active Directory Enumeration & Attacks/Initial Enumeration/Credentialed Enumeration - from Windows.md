## Credentialed Enumeration from Windows

This section covers enumeration from a domain-joined Windows attack host using the ActiveDirectory PowerShell module, [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1), [SharpView](https://github.com/dmchell/SharpView), [Snaffler](https://github.com/SnaffCon/Snaffler), and [SharpHound](https://github.com/BloodHoundAD/SharpHound)/[BloodHound](https://github.com/BloodHoundAD/BloodHound). Not every finding here will directly lead to an attack path. Some will be informational and still worth including in your report to help the client improve their security posture.

***

## ActiveDirectory PowerShell Module

The ActiveDirectory module ships with 147 cmdlets and can be a stealthier option than dropping a third-party tool onto a host, since its usage can blend in with normal administrative activity.

### Loading the Module

Check which modules are currently loaded, then import the ActiveDirectory module if it is not already present:

```powershell
Get-Module
Import-Module ActiveDirectory
Get-Module
```

### Get Domain Info

`Get-ADDomain` returns the domain SID, functional level, child domains, FSMO role holders, and more:

```powershell
Get-ADDomain
```

Key fields from the output:

```
ChildDomains    : {LOGISTICS.INLANEFREIGHT.LOCAL}
DomainMode      : Windows2016Domain
DomainSID       : S-1-5-21-3842939050-3880317879-2865463114
PDCEmulator     : ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
Forest          : INLANEFREIGHT.LOCAL
```

### Find Kerberoastable Accounts

Filter for user accounts with a `ServicePrincipalName` (SPN) set. These accounts are candidates for Kerberoasting:

```powershell
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
```

Example output:

```
SamAccountName       : adfs
ServicePrincipalName : {adfsconnect/azure01.inlanefreight.local}

SamAccountName       : backupagent
ServicePrincipalName : {backupjob/veam001.inlanefreight.local}
```

### Check Domain Trust Relationships

`Get-ADTrust` returns all domain trust relationships, including whether they are intra-forest or forest-transitive and which direction they flow:

```powershell
Get-ADTrust -Filter *
```

Example output:

```
Name                 : LOGISTICS.INLANEFREIGHT.LOCAL
Direction            : BiDirectional
IntraForest          : True
ForestTransitive     : False

Name                 : FREIGHTLOGISTICS.LOCAL
Direction            : BiDirectional
IntraForest          : False
ForestTransitive     : True
```

### Group Enumeration

List all groups in the domain, then query a specific group by name for detail and membership:

```powershell
Get-ADGroup -Filter * | select name
Get-ADGroup -Identity "Backup Operators"
Get-ADGroupMember -Identity "Backup Operators"
```

Example membership output:

```
SamAccountName    : backupagent
objectClass       : user
SID               : S-1-5-21-3842939050-3880317879-2865463114-5220
```

Membership in `Backup Operators` is worth noting as it can be abused to take over the domain.

***

## PowerView

[PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) is a PowerShell recon tool for Active Directory that can enumerate users, groups, computers, ACLs, GPOs, trusts, shares, sessions, and SPN accounts. It requires more manual effort than BloodHound but surfaces subtle misconfigurations that automated tools may miss.

### Key Functions Reference

| Function | Description |
|----------|-------------|
| `Get-Domain` | Returns AD object for the current or specified domain |
| `Get-DomainUser` | Returns all or specific user objects |
| `Get-DomainGroup` | Returns all or specific group objects |
| `Get-DomainComputer` | Returns all or specific computer objects |
| `Get-DomainGroupMember` | Returns members of a specific domain group |
| `Get-DomainTrustMapping` | Enumerates all trusts for the current domain and any others seen |
| `Get-DomainOU` | Searches for all or specific OU objects |
| `Get-DomainGPO` | Returns all or specific GPO objects |
| `Get-DomainPolicy` | Returns the default domain or DC password policy |
| `Find-DomainUserLocation` | Finds machines where specific users are logged in |
| `Find-InterestingDomainAcl` | Finds ACLs with modification rights set to non-built-in objects |
| `Find-LocalAdminAccess` | Finds machines where the current user has local admin rights |
| `Get-DomainTrust` | Returns domain trusts for the current or specified domain |
| `Get-ForestTrust` | Returns all forest trusts |
| `Test-AdminAccess` | Tests if the current user has admin access to a local or remote machine |
| `Get-DomainSPNTicket` | Requests the Kerberos ticket for a specified SPN account |
| `Export-PowerViewCSV` | Appends results to a CSV file |

### Detailed User Information

Pull a rich set of attributes for a specific user, including group membership, password age, SPN, and UAC flags:

```powershell
Get-DomainUser -Identity mmorgan -Domain inlanefreight.local | Select-Object -Property name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol
```

Notable fields from the output:

```
name               : Matthew Morgan
admincount         : 1
useraccountcontrol : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD, DONT_REQ_PREAUTH
memberof           : {VPN Users, Shared Calendar Read, Printer Access, File Share H Drive...}
```

`DONT_REQ_PREAUTH` indicates AS-REP Roasting may be possible against this account.

### Recursive Group Membership

Using `-Recurse` with `Get-DomainGroupMember` reveals users who inherit group membership through nested groups. This is important because a user in a nested group may have Domain Admin rights that are not immediately obvious:

```powershell
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
```

Example output showing `Secadmins` nesting into `Domain Admins`:

```
GroupName    : Domain Admins
MemberName   : svc_qualys

GroupName    : Secadmins
MemberName   : spong1990
```

Both `svc_qualys` and `spong1990` (via `Secadmins`) effectively hold Domain Admin rights.

### Trust Mapping

```powershell
Get-DomainTrustMapping
```

Example output:

```
SourceName    : INLANEFREIGHT.LOCAL
TargetName    : LOGISTICS.INLANEFREIGHT.LOCAL
TrustAttributes : WITHIN_FOREST
TrustDirection  : Bidirectional

SourceName    : INLANEFREIGHT.LOCAL
TargetName    : FREIGHTLOGISTICS.LOCAL
TrustAttributes : FOREST_TRANSITIVE
TrustDirection  : Bidirectional
```

### Test Local Admin Access

```powershell
Test-AdminAccess -ComputerName ACADEMY-EA-MS01
```

```
ComputerName     IsAdmin
------------     -------
ACADEMY-EA-MS01  True
```

### Find SPN Accounts for Kerberoasting

```powershell
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
```

Example output:

```
serviceprincipalname                           samaccountname
--------------------                           --------------
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433  sqldev
MSSQLSvc/SPSJDB.inlanefreight.local:1433       sqlprod
backupjob/veam001.inlanefreight.local          backupagent
```

***

## SharpView

[SharpView](https://github.com/dmchell/SharpView) is a .NET port of PowerView. It supports most of the same functions and is useful when a client has hardened against PowerShell usage or when you need to avoid PowerShell-based detection. Pass `-Help` to any method to see its argument list:

```powershell
.\SharpView.exe Get-DomainUser -Help
.\SharpView.exe Get-DomainUser -Identity forend
```

SharpView returns the same attribute set as PowerView, including group memberships, logon timestamps, bad password counts, and UAC flags.

***

## Snaffler

[Snaffler](https://github.com/SnaffCon/Snaffler) automates the process of hunting domain shares for sensitive files. It enumerates all hosts in the domain, lists readable shares, and iterates through accessible directories looking for files of interest such as key files, database dumps, credential stores, and config files. It must be run from a domain-joined host or in a domain-user context.

```powershell
.\Snaffler.exe -s -d inlanefreight.local -o snaffler.log -v data
```

Flags used:

- `-s` prints results to the console
- `-d` specifies the domain to search
- `-o` writes results to a log file for later review
- `-v data` sets verbosity to show only results, keeping the output clean

Snaffler colour codes its output. Example hits from a run:

```
[File] {Red}  Department Shares\IT\Infosec\ShowReset.key
[File] {Red}  Department Shares\IT\Development\DenyRedo.sqldump
[File] {Black} Department Shares\IT\Infosec\GroupBackup.kdb
[File] {Black} Department Shares\IT\Infosec\StopTrace.ppk
```

Red entries are highest priority. Always output to a log file and review it after the run. Raw Snaffler output is also useful as supplemental data for clients so they can prioritise which shares to lock down first.

***

## SharpHound and BloodHound

[SharpHound](https://github.com/BloodHoundAD/SharpHound) is the C# collector for [BloodHound](https://github.com/BloodHoundAD/BloodHound). It pulls data from the domain including users, groups, computers, sessions, ACLs, GPOs, trusts, local admin rights, RDP, DCOM, WinRM access, and SPN targets, then packages it as a zip file for ingestion into the BloodHound GUI.

### Running SharpHound

```powershell
.\SharpHound.exe -c All --zipfilename ILFREIGHT
```

Example output:

```
Resolved Collection Methods: Group, LocalAdmin, GPOLocalGroup, Session, LoggedOn, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
Enumeration finished in 00:01:22.7919186
Status: 3809 objects finished
```

### Loading Data into BloodHound

1. Launch BloodHound: type `bloodhound` in CMD or PowerShell
2. Authenticate with `neo4j : HTB_@cademy_stdnt!`
3. Click Upload Data on the right-hand side
4. Select the generated zip file and click Open
5. Wait for all JSON files to show 100% complete in the Upload Progress window

### Useful Pre-Built Queries

- **Find Shortest Paths to Domain Admins**: maps all attack paths from any compromised account to Domain Admin
- **Find Computers with Unsupported Operating Systems**: surfaces Windows 7, Server 2003, Server 2008 and similar legacy hosts that may be vulnerable to older exploits like MS08-067
- **Find Computers where Domain Users are Local Admin**: identifies any host where every domain user has local admin rights, making it accessible to any account you control

Legacy hosts found through the unsupported OS query should be validated as live before reporting. If they are still active, recommend network segmentation and decommission planning. If they are stale AD records, recommend clean-up.

Keep detailed notes throughout the engagement. Log every file transferred to and from domain hosts, including where it was placed on disk. Clean up all tools and artefacts at the conclusion of the assessment.
