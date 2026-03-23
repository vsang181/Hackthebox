# Password Spraying - Building a Target User List

Before spraying, you need a valid list of domain users. The quality and completeness of that list directly affects your chances of success. Below are the main methods for building one, ordered from most reliable to least.

***

## Methods for Gathering Users

- Leverage an SMB NULL session to pull a complete user list from a Domain Controller
- Use an LDAP anonymous bind to query Active Directory without credentials
- Use [Kerbrute](https://github.com/ropnop/kerbrute) to validate usernames against a wordlist such as [statistically-likely-usernames](https://github.com/insidetrust/statistically-likely-usernames)
- Use valid credentials obtained via LLMNR/NBT-NS poisoning with [Responder](https://github.com/lgandx/Responder), a previous spray, or provided by the client
- Fall back to external sources such as email harvesting or [linkedin2username](https://github.com/initstring/linkedin2username) when internal methods fail

Always log every spray attempt. Record the accounts targeted, the Domain Controller used, the time and date, and the passwords attempted. If an account lockout occurs or the client notices suspicious logon attempts, your logs allow you to cross-reference against their SIEM and confirm the activity was part of the test.

***

## SMB NULL Session - User Enumeration

### Using enum4linux

[enum4linux](https://github.com/portcullislabs/enum4linux) with the `-U` flag retrieves the user list, which you then pipe through `grep` and `cut` to produce a clean one-per-line output:

```bash
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
```

Example output:

```
administrator
guest
krbtgt
lab_adm
htb-student
avazquez
pfalcon
fanthony
```

### Using rpcclient

Connect anonymously with [rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html) and run the `enumdomusers` command:

```bash
rpcclient -U "" -N 172.16.5.5
```

```
rpcclient $> enumdomusers
user:[administrator] rid:[0x1f4]
user:[guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[lab_adm] rid:[0x3e9]
user:[htb-student] rid:[0x457]
user:[avazquez] rid:[0x458]
```

### Using CrackMapExec

[CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) with the `--users` flag is particularly useful because it also returns `badpwdcount` and `baddpwdtime` for each account. Use this data to remove accounts that are close to the lockout threshold before spraying.

```bash
crackmapexec smb 172.16.5.5 --users
```

Example output:

```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\administrator  badpwdcount: 0 baddpwdtime: 2022-01-10 13:23:09
SMB  172.16.5.5  445  ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\avazquez       badpwdcount: 0 baddpwdtime: 2022-02-17 22:59:22
SMB  172.16.5.5  445  ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\htb-student    badpwdcount: 0 baddpwdtime: 2022-02-22 14:48:26
```

Note on `badpwdcount`: In environments with multiple Domain Controllers, this value is tracked separately on each one. To get an accurate total, query each DC individually and sum the values, or query the DC holding the PDC Emulator FSMO role.

***

## LDAP Anonymous Bind - User Enumeration

### Using ldapsearch

[ldapsearch](https://linux.die.net/man/1/ldapsearch) requires a valid LDAP search filter. The following command filters for user objects and extracts just the `sAMAccountName` values:

```bash
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
```

Example output:

```
guest
ACADEMY-EA-DC01$
ACADEMY-EA-MS01$
htb-student
avazquez
pfalcon
fanthony
```

Note: In newer versions of ldapsearch, replace `-h` with `-H`.

### Using windapsearch

[windapsearch](https://github.com/ropnop/windapsearch) simplifies anonymous LDAP enumeration. Pass a blank username with `-u` and use `-U` to return only user objects:

```bash
./windapsearch.py --dc-ip 172.16.5.5 -u "" -U
```

Example output:

```
[+] No username provided. Will try anonymous bind.
[+] Found 2906 users:

cn: Htb Student
userPrincipalName: htb-student@inlanefreight.local

cn: Annie Vazquez
userPrincipalName: avazquez@inlanefreight.local

cn: Paul Falcon
userPrincipalName: pfalcon@inlanefreight.local
```

***

## Kerbrute - Username Enumeration

[Kerbrute](https://github.com/ropnop/kerbrute) uses [Kerberos Pre-Authentication](https://ldapwiki.com/wiki/Wiki.jsp?page=Kerberos%20Pre-Authentication) to validate usernames. It sends TGT requests to the KDC without supplying pre-auth data. A response of `PRINCIPAL UNKNOWN` means the username is invalid. A prompt for pre-authentication means the username exists and is marked valid.

This method does not generate Windows Event ID [4625 (An account failed to log on)](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4625), making it quieter than traditional authentication attempts. However, once you switch from enumeration to spraying with Kerbrute, failed pre-auth attempts do count toward the account lockout threshold.

Run enumeration using the [jsmith.txt](https://github.com/insidetrust/statistically-likely-usernames/blob/master/jsmith.txt) wordlist of 48,705 common usernames in `flast` format:

```bash
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
```

Example output:

```
[+] VALID USERNAME: jjones@inlanefreight.local
[+] VALID USERNAME: sbrown@inlanefreight.local
[+] VALID USERNAME: tjohnson@inlanefreight.local
[+] VALID USERNAME: jwilson@inlanefreight.local
[+] VALID USERNAME: asanchez@inlanefreight.local
[+] VALID USERNAME: dlewis@inlanefreight.local
```

Kerbrute checked over 48,000 usernames in roughly 12 seconds and returned 50+ valid accounts. This method does generate Event ID [4768 (A Kerberos TGT was requested)](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4768), but only if [Kerberos event logging](https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/enable-kerberos-event-logging) is enabled via Group Policy. An influx of this event ID is a detection opportunity for defenders and worth noting as a recommendation in your report.

***

## Credentialed Enumeration

With valid credentials, [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) with `--users` remains the fastest option and continues to surface `badpwdcount` data:

```bash
sudo crackmapexec smb 172.16.5.5 -u htb-student -p Academy_student_AD! --users
```

Example output:

```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\administrator  badpwdcount: 1 baddpwdtime: 2022-02-23 21:43:35
SMB  172.16.5.5  445  ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\avazquez       badpwdcount: 20 baddpwdtime: 2022-02-17 22:59:22
SMB  172.16.5.5  445  ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\pfalcon        badpwdcount: 0 baddpwdtime: 1600-12-31 19:03:58
```

Notice `avazquez` has a `badpwdcount` of 20. This account should be excluded from any spray to avoid triggering a lockout.

***

## Fallback - External Enumeration

If no internal method is available, external sources can still produce a usable list:

- Scrape company email addresses from public breach data or job postings
- Use [linkedin2username](https://github.com/initstring/linkedin2username) to generate probable usernames from a company's LinkedIn employee list
- Cross-reference with the [statistically-likely-usernames](https://github.com/insidetrust/statistically-likely-usernames) repo to format names into common AD username patterns

External lists will not be as complete as those pulled directly from AD, but they can still be enough to obtain initial access.

***

## Using SYSTEM Access

If you have SYSTEM-level access on a Windows host but no valid domain credentials, you can still query Active Directory. The SYSTEM account can impersonate the machine's computer object, and computer accounts authenticate to the domain like user accounts. This allows LDAP and RPC queries to succeed even without an explicit domain user credential.
