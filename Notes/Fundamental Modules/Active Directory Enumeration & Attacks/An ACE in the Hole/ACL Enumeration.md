# ACL Enumeration

Enumerating ACLs effectively requires a targeted approach. Running `Find-InterestingDomainAcl` against the entire domain returns an unmanageable volume of data. The correct method is to start from a user you control, get their SID, and trace the attack path forward one hop at a time.

***

## Targeted Enumeration with PowerView

### Step 1: Get the SID of Your Controlled User

```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid wley
```

### Step 2: Find Objects the User Has Rights Over

Without `-ResolveGUIDs`, the `ObjectAceType` field returns a raw GUID that is not human-readable:

```powershell
Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

Example output:

```
ObjectDN              : CN=Dana Amundsen,OU=DevOps,...,DC=INLANEFREIGHT,DC=LOCAL
ActiveDirectoryRights : ExtendedRight
ObjectAceType         : 00299570-246d-11d0-a768-00aa006e0529
AceQualifier          : AccessAllowed
SecurityIdentifier    : S-1-5-21-...-1181
```

To decode a GUID manually, query the Extended-Rights container:

```powershell
$guid = "00299570-246d-11d0-a768-00aa006e0529"
Get-ADObject -SearchBase "CN=Extended-Rights,$((Get-ADRootDSE).ConfigurationNamingContext)" -Filter {ObjectClass -like 'ControlAccessRight'} -Properties * | Select Name,DisplayName,DistinguishedName,rightsGuid | ?{$_.rightsGuid -eq $guid} | fl
```

Output:

```
Name        : User-Force-Change-Password
DisplayName : Reset Password
rightsGuid  : 00299570-246d-11d0-a768-00aa006e0529
```

Always use `-ResolveGUIDs` in practice to skip this manual lookup. The same search with the flag shows the result immediately in human-readable form:

```powershell
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

Output:

```
ObjectDN              : CN=Dana Amundsen,...
ActiveDirectoryRights : ExtendedRight
ObjectAceType         : User-Force-Change-Password
AceQualifier          : AccessAllowed
```

`wley` has `User-Force-Change-Password` over `damundsen`.

***

## Without PowerView: Built-in Cmdlets

If PowerView is blocked, you can enumerate ACLs using native `Get-Acl` and `Get-ADUser` cmdlets. This is slower but valuable when working from a client's managed machine with no ability to bring in external tools.

First, generate a list of all domain users:

```powershell
Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
```

Then iterate through each user and filter for ACEs tied to your controlled account:

```powershell
foreach($line in [System.IO.File]::ReadLines("C:\Users\htb-student\Desktop\ad_users.txt")) {
    get-acl "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match 'INLANEFREIGHT\\wley'}
}
```

Output:

```
Path                  : ...CN=Dana Amundsen,...
ActiveDirectoryRights : ExtendedRight
ObjectType            : 00299570-246d-11d0-a768-00aa006e0529
AccessControlType     : Allow
IdentityReference     : INLANEFREIGHT\wley
```

The `ObjectType` GUID is the same one from the previous step. You would then decode it the same way.

***

## Tracing the Full Attack Chain

Each hop in the chain is enumerated by taking the next account's SID and running the same query.

### damundsen's Rights

```powershell
$sid2 = Convert-NameToSid damundsen
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid2} -Verbose
```

Output:

```
ObjectDN              : CN=Help Desk Level 1,OU=Security Groups,...
ActiveDirectoryRights : ListChildren, ReadProperty, GenericWrite
AceQualifier          : AccessAllowed
```

`damundsen` has `GenericWrite` over the `Help Desk Level 1` group, meaning we can add any user to that group.

### Check for Nested Group Membership

```powershell
Get-DomainGroup -Identity "Help Desk Level 1" | select memberof
```

Output:

```
memberof
--------
CN=Information Technology,OU=Security Groups,...,DC=INLANEFREIGHT,DC=LOCAL
```

`Help Desk Level 1` is nested inside `Information Technology`. Any user added to `Help Desk Level 1` inherits the rights of `Information Technology`.

### Information Technology Group's Rights

```powershell
$itgroupsid = Convert-NameToSid "Information Technology"
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $itgroupsid} -Verbose
```

Output:

```
ObjectDN              : CN=Angela Dunn,OU=Server Admin,...
ActiveDirectoryRights : GenericAll
AceQualifier          : AccessAllowed
```

The `Information Technology` group has `GenericAll` over `adunn`. This allows modifying group membership, forcing a password change, or performing a targeted Kerberoasting attack.

### adunn's Rights

```powershell
$adunnsid = Convert-NameToSid adunn
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $adunnsid} -Verbose
```

Output:

```
ObjectDN              : DC=INLANEFREIGHT,DC=LOCAL
ObjectAceType         : DS-Replication-Get-Changes-In-Filtered-Set

ObjectDN              : DC=INLANEFREIGHT,DC=LOCAL
ObjectAceType         : DS-Replication-Get-Changes
```

`adunn` holds the two replication rights needed to perform a **DCSync attack**, which can dump all password hashes in the domain.

***

## Full Attack Chain Summary

| Step | User | Right | Target |
|------|------|-------|--------|
| 1 | `wley` | `ForceChangePassword` | `damundsen` |
| 2 | `damundsen` | `GenericWrite` | `Help Desk Level 1` group |
| 3 | `Help Desk Level 1` | Nested membership | `Information Technology` group |
| 4 | `Information Technology` | `GenericAll` | `adunn` |
| 5 | `adunn` | `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-In-Filtered-Set` | Domain (DCSync) |

***

## Enumerating with BloodHound

[BloodHound](https://github.com/BloodHoundAD/BloodHound) maps the same attack chain visually in seconds. After uploading data collected by the [SharpHound](https://github.com/BloodHoundAD/SharpHound) ingestor:

1. Set `wley` as the starting node
2. Go to the `Node Info` tab and scroll to `Outbound Control Rights`
3. `First Degree Object Control` shows direct rights, `1` entry for `ForceChangePassword` over `damundsen`
4. `Transitive Object Control` shows the full chained path, `16` objects reachable through the chain

Right-clicking on any edge in the graph and selecting `Help` provides:

- A description of the specific right being abused
- The tools and exact commands to exploit it
- Operational security considerations
- External references for further reading

BloodHound also includes pre-built queries such as "Find Principals with DCSync Rights" that will confirm `adunn`'s replication rights against the domain object directly, without manual enumeration.

***

## Why Walk Through the Manual Method?

Tools can fail, be blocked by EDR, or be unavailable on a client-managed host. Understanding what `Get-DomainObjectACL` is doing under the hood, how GUIDs map to human-readable right names, and how to replicate the enumeration with native cmdlets gives you fallback options when automation is not available. Always understand what your tools are doing before relying on them.
