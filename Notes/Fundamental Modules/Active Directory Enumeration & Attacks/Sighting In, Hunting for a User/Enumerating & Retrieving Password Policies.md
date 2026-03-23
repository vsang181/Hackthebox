# Enumerating and Retrieving Password Policies

There are several ways to pull the domain password policy depending on whether you hold valid credentials and what your position is on the network. The methods below cover both Linux and Windows attack paths.

***

## From Linux - Credentialed

With valid domain credentials, [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) makes retrieval straightforward using the `--pass-pol` flag:

```bash
crackmapexec smb 172.16.5.5 -u avazquez -p Password123 --pass-pol
```

Example output:

```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Minimum password length: 8
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Password history length: 24
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Maximum password age: Not Set
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Password Complexity Flags: 000001
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Account Lockout Threshold: 5
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Locked Account Duration: 30 minutes
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Reset Account Lockout Counter: 30 minutes
```

***

## From Linux - SMB NULL Sessions

Without credentials, SMB NULL sessions may allow unauthenticated retrieval of domain information including users, groups, computers, account attributes, and the password policy. This misconfiguration is common on domains where older Domain Controllers were upgraded in place, carrying insecure legacy defaults forward.

### Using rpcclient

[rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html) can connect to a Domain Controller using a NULL session and issue RPC commands to pull domain information:

```bash
rpcclient -U "" -N 172.16.5.5
```

Once connected, run `querydominfo` to confirm NULL session access and then `getdompwinfo` to retrieve the password policy:

```
rpcclient $> querydominfo
Domain:         INLANEFREIGHT
Total Users:    3650
Server Role:    ROLE_DOMAIN_PDC

rpcclient $> getdompwinfo
min_password_length: 8
password_properties: 0x00000001
    DOMAIN_PASSWORD_COMPLEX
```

### Using enum4linux

[enum4linux](https://labs.portcullis.co.uk/tools/enum4linux) wraps the [Samba suite of tools](https://www.samba.org/samba/docs/current/man-html/samba.7.html) (nmblookup, net, rpcclient, smbclient) into a single enumeration tool. It comes pre-installed on many pentesting distributions including Parrot Security Linux.

Common tool and port reference:

| Tool | Ports |
|------|-------|
| nmblookup | 137/UDP |
| nbtstat | 137/UDP |
| net | 139/TCP, 135/TCP, 49152-65535 TCP/UDP |
| rpcclient | 135/TCP |
| smbclient | 445/TCP |

Pull the password policy with:

```bash
enum4linux -P 172.16.5.5
```

Example output:

```
[+] Minimum password length: 8
[+] Password history length: 24
[+] Maximum password age: Not Set
[+] Password Complexity Flags: 000001
[+] Minimum password age: 1 day 4 minutes
[+] Reset Account Lockout Counter: 30 minutes
[+] Locked Account Duration: 30 minutes
[+] Account Lockout Threshold: 5
[+] Forced Log off Time: Not Set
```

### Using enum4linux-ng

[enum4linux-ng](https://github.com/cddmp/enum4linux-ng) is a Python rewrite of enum4linux with additional features including JSON and YAML export, coloured output, and cleaner structured results. Use the `-oA` flag to write output files:

```bash
enum4linux-ng -P 172.16.5.5 -oA ilfreight
```

The `-oA` flag generates both `ilfreight.json` and `ilfreight.yaml`. Inspect the JSON output with:

```bash
cat ilfreight.json
```

The JSON output includes SMB dialect information, NULL session confirmation, and a cleanly structured password policy block:

```json
domain_password_information:
  pw_history_length: 24
  min_pw_length: 8
  min_pw_age: 1 day 4 minutes
  max_pw_age: not set
  pw_properties:
  - DOMAIN_PASSWORD_COMPLEX: true
  - DOMAIN_PASSWORD_LOCKOUT_ADMINS: false
domain_lockout_information:
  lockout_observation_window: 30 minutes
  lockout_duration: 30 minutes
  lockout_threshold: 5
```

***

## From Linux - LDAP Anonymous Bind

[LDAP anonymous binds](https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/anonymous-ldap-operations-active-directory-disabled) are a legacy configuration that allows unauthenticated retrieval of users, groups, computers, account attributes, and the password policy. As of Windows Server 2003, only authenticated users can initiate LDAP requests, but misconfigurations still appear in the wild when an admin has set up an application requiring anonymous access and granted broader permissions than intended.

Tools that support LDAP anonymous enumeration include [windapsearch](https://github.com/ropnop/windapsearch), [ldapsearch](https://linux.die.net/man/1/ldapsearch), and ad-ldapdomaindump.py.

Using `ldapsearch` to pull the password policy:

```bash
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

Example output:

```
lockoutThreshold: 5
maxPwdAge: -9223372036854775808
minPwdAge: -864000000000
minPwdLength: 8
pwdProperties: 1
pwdHistoryLength: 24
```

Note: In newer versions of ldapsearch, replace `-h` with `-H`.

***

## From Windows - NULL Session

Establishing an SMB NULL session from a Windows host is less common but achievable using `net use`:

```cmd
net use \\DC01\ipc$ "" /u:""
```

Common error codes when testing authentication via this method:

```cmd
# Account disabled
net use \\DC01\ipc$ "" /u:guest
System error 1331 - This user can't sign in because this account is currently disabled.

# Wrong password
net use \\DC01\ipc$ "password" /u:guest
System error 1326 - The user name or password is incorrect.

# Account locked out
net use \\DC01\ipc$ "password" /u:guest
System error 1909 - The referenced account is currently locked out and may not be logged on to.
```

***

## From Windows - Credentialed

With domain access from a Windows host, the built-in `net.exe` binary, [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1), [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec), and other tools can all retrieve the policy.

### Using net.exe

No additional tools needed, useful when you cannot transfer files to the target:

```cmd
net accounts
```

Example output:

```
Minimum password age (days):              1
Maximum password age (days):              Unlimited
Minimum password length:                  8
Length of password history maintained:    24
Lockout threshold:                        5
Lockout duration (minutes):               30
Lockout observation window (minutes):     30
```

### Using PowerView

[PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) returns the same data with additional detail such as Kerberos policy and registry paths:

```powershell
Import-Module .\PowerView.ps1
Get-DomainPolicy
```

Example output:

```
SystemAccess : @{MinimumPasswordAge=1; MaximumPasswordAge=-1; MinimumPasswordLength=8;
               PasswordComplexity=1; PasswordHistorySize=24; LockoutBadCount=5;
               ResetLockoutCount=30; LockoutDuration=30; ClearTextPassword=0}
KerberosPolicy : @{MaxTicketAge=10; MaxRenewAge=7; MaxClockSkew=5}
```

`PasswordComplexity=1` confirms complexity is enforced, which `net accounts` does not directly surface.

***

## Analysing the Policy

Breaking down the INLANEFREIGHT.LOCAL policy retrieved above:

- Minimum password length is 8, which is low. Modern organisations often enforce 10-14 characters
- Lockout threshold is 5 bad attempts. Some organisations set this as low as 3 or not at all
- Lockout duration is 30 minutes with automatic unlock, meaning no admin intervention is needed to recover a locked account
- Password complexity is enabled, requiring 3 of 4 character types: uppercase, lowercase, number, special character. Passwords like `Welcome1` satisfy this requirement but are still trivially weak
- Passwords never expire, increasing the likelihood that weak or default passwords remain in use

Default Windows domain password policy values for reference:

| Policy | Default Value |
|--------|--------------|
| Enforce password history | 24 days |
| Maximum password age | 42 days |
| Minimum password age | 1 day |
| Minimum password length | 7 |
| Password complexity | Enabled |
| Reversible encryption | Disabled |
| Account lockout duration | Not set |
| Account lockout threshold | 0 |
| Reset lockout counter after | Not set |

***
