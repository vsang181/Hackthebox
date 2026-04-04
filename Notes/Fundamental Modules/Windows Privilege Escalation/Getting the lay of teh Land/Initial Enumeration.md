## Initial Enumeration

Initial enumeration on a Windows host is about building a complete picture before touching any exploit. Every command below answers a specific question that either rules out or confirms a privilege escalation path. 

***

## System Information

### OS Version and Patch Level

The first thing to capture is the OS name, build number, and installed hotfixes. This directly maps to known CVEs and rules out or confirms kernel-level exploits. 

```cmd
:: Full system details in one shot
systeminfo

:: Targeted one-liner for OS info only
systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /C:"System Type" /C:"System Boot Time"

:: Boot time check - if the system hasn't rebooted in 6+ months, it probably hasn't been patched
:: "System Boot Time: 10/1/2020" in 2021 = almost certainly unpatched
```

Reading key fields from `systeminfo`:

```
OS Name:       Microsoft Windows Server 2016 Standard   <- maps to specific CVEs
OS Version:    10.0.14393 Build 14393                   <- exact build for WES-NG
System Model:  VMware7,1                                <- confirms VM, affects some exploits
Domain:        WORKGROUP                                <- not domain-joined
Hotfix(s):     3 Hotfix(s) Installed                   <- very few patches = high value target
```

### Patch Enumeration

```cmd
:: cmd.exe method
wmic qfe get Caption, HotFixID, InstalledBy, InstalledOn

:: PowerShell method (cleaner output)
Get-HotFix | ft -AutoSize
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10

:: Feed to WES-NG on your attack machine
systeminfo > C:\Windows\Temp\sysinfo.txt
:: Transfer file, then: python3 wes.py sysinfo.txt --impact "Elevation of Privilege"
```

***

## Running Processes and Services

```cmd
:: Processes with associated service names - essential for spotting non-standard services
tasklist /svc

:: Filter for processes not running under standard system accounts
:: Look for: custom service names, third-party software (FileZilla, XAMPP, etc.)
tasklist /svc | findstr /v "svchost\|System\|smss\|csrss\|wininit\|winlogon\|services\|lsass\|dwm"

:: PowerShell - more detail including path to executable
Get-Process | Select-Object Name, Id, Path | Sort-Object Name

:: Services running as SYSTEM or specific users (high value targets)
wmic service get name, startname, pathname, startmode | findstr /i "system\|admin"
Get-WmiObject Win32_Service | Select-Object Name, StartName, PathName, StartMode |
    Where-Object {$_.StartName -match "SYSTEM|Administrator|Admin"} | ft -AutoSize
```

Things to hunt for in process output: 

```
FileZilla Server.exe   -> FTP server, check version for CVE, check for anonymous access
inetinfo.exe           -> IIS admin service
MsMpEng.exe            -> Windows Defender running (affects tool usage)
vmtoolsd.exe           -> Confirms VMware (useful context)
spoolsv.exe (Spooler)  -> Print Spooler service, check for PrintNightmare
Any .exe in C:\Users\  -> User-installed software running as a service
```

***

## Environment Variables

```cmd
:: Full environment dump
set

:: Key variables to analyse manually:
:: PATH    - custom directories? writable directories added?
:: HOMEDRIVE/HOMEPATH - is home on a network share?
:: LOGONSERVER - which DC authenticated you
:: USERDOMAIN - confirms domain membership
```

What to hunt for in the `PATH` variable: 
```
Normal:  C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;...

Dangerous additions:
  C:\Python27\           <- if writable, DLL hijacking possible
  C:\Tools\              <- custom directory, check permissions
  C:\Temp\               <- writable by users, PATH hijacking opportunity
  .                      <- current directory in PATH = trivially exploitable

Rule: Writable directory on the LEFT side of PATH = more dangerous than on the right,
because Windows searches left to right and takes the first match found.
```

Roaming Profile abuse via `USERPROFILE` and `APPDATA`:

```cmd
:: If HOMEDRIVE points to a share (\\server\share\username), credentials may be cached there
:: Startup folder persistence applies across all machines the user logs into:
dir "%USERPROFILE%\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup"
```

***

## Installed Software

```cmd
:: WMI method (shows all MSI-installed software)
wmic product get name, version, vendor

:: PowerShell - more reliable, also catches non-MSI installs
Get-WmiObject -Class Win32_Product | Select-Object Name, Version | Sort-Object Name

:: Also check these locations for non-MSI installs
Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Select-Object DisplayName, DisplayVersion | Sort-Object DisplayName
Get-ItemProperty HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Select-Object DisplayName, DisplayVersion | Sort-Object DisplayName
```

Interesting software findings: 

```
Java (old version)          -> check CVE database for specific version
FileZilla / WinSCP / PuTTY  -> run SessionGopher/LaZagne for saved credentials
XAMPP / WAMP / Apache       -> check for writable web root, config files with DB passwords
SQL Server                  -> check for xp_cmdshell, SA with blank password
Any custom/internal software -> check service binary permissions, check for DLL hijacking
```

***

## Listening Services and Ports

```cmd
:: All active connections and listening ports with PIDs
netstat -ano

:: Map PID to process name
tasklist /svc /FI "PID eq <pid>"

:: PowerShell equivalent
Get-NetTCPConnection | Select-Object LocalAddress, LocalPort, State, OwningProcess |
    Sort-Object LocalPort
```

Reading `netstat -ano` output - what to look for: 
```
0.0.0.0:21     PID 1096   <- FTP listening externally, check for anon access
0.0.0.0:1433   PID 3520   <- SQL Server, try default creds
127.0.0.1:8080 PID 4444   <- internal web app not visible externally, only accessible post-foothold
0.0.0.0:3389              <- RDP enabled, useful for lateral movement after credential theft
```

***

## User and Group Enumeration

### Current Session Context

```cmd
:: Who are you running as
whoami

:: Full token: user SID, all group memberships, all privileges
whoami /all

:: Just privileges (most important for escalation paths)
whoami /priv

:: Just groups (look for Backup Operators, Event Log Readers, DnsAdmins, etc.)
whoami /groups

:: Who is currently logged into the system (spot active admin sessions)
query user
qwinsta
```

Privilege flags to look for immediately in `whoami /priv`: 

```
SeImpersonatePrivilege          -> Potato attacks (JuicyPotato, RoguePotato, PrintSpoofer)
SeAssignPrimaryTokenPrivilege   -> Same as above, alternative path
SeBackupPrivilege               -> Read any file regardless of ACL, dump SAM/SYSTEM
SeRestorePrivilege              -> Write any file, replace binaries
SeDebugPrivilege                -> Attach to LSASS, dump credentials
SeTakeOwnershipPrivilege        -> Take ownership of any object, then modify
SeLoadDriverPrivilege           -> Load kernel driver -> kernel-level code execution
SeCreateSymbolicLinkPrivilege   -> Create symlinks, useful in combination with other bugs
```

### System Users

```cmd
:: List all local user accounts
net user

:: Detailed info on a specific user (last login, group memberships, password policy)
net user <username>

:: PowerShell alternative
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordLastSet

:: Check home directories for interesting files
dir C:\Users
```

### Groups

```cmd
:: All local groups
net localgroup

:: Members of the local Administrators group (priority target)
net localgroup Administrators

:: Other privileged groups worth checking
net localgroup "Backup Operators"
net localgroup "Remote Desktop Users"
net localgroup "Remote Management Users"
net localgroup "Event Log Readers"

:: PowerShell
Get-LocalGroupMember Administrators | ft Name, PrincipalSource
```

### Password Policy

```cmd
:: Reveals lockout threshold, minimum length, max age
:: Minimum password length of 0 = blank passwords may exist
:: Lockout threshold of Never = bruteforce is safe
net accounts
```

***

## Full Initial Enumeration One-Liner Sequence

Run this sequence immediately after landing on a new host. It covers every key data point discussed:

```cmd
:: System
systeminfo & echo. & wmic qfe get HotFixID, InstalledOn

:: Current user context
whoami /all

:: Users and groups
net user & echo. & net localgroup & echo. & net localgroup Administrators

:: Password policy
net accounts

:: Running processes (spot interesting services)
tasklist /svc

:: Network connections (spot internal-only services)
netstat -ano

:: Environment (check PATH for writable custom directories)
set

:: Installed software
wmic product get name, version
```
