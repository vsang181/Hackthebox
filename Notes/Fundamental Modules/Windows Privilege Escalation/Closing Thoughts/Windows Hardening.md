## Windows Hardening

Hardening is the most effective way to eliminate or reduce the attack surface that this entire module covers. Every misconfiguration, weak permission, and missing patch explored in previous sections maps directly to a hardening control. 

***

## 1. Secure OS Baseline

Build a clean, standardised image before deploying any host. This removes bloatware, pre-configures security settings, and ensures every machine in the environment starts from an identical, known-good state. 
```
Tooling for image management:
- Windows Deployment Services (WDS)        -> PXE boot image deployment
- System Center Configuration Manager (SCCM/ConfigMgr) -> enterprise push
- Microsoft Deployment Toolkit (MDT)       -> free, lighter alternative

What the image should include before deployment:
- All required business applications (no extras)
- Latest cumulative + security updates pre-applied
- Baseline Group Policy settings baked in
- BitLocker pre-staged if hardware supports TPM
- WDigest disabled (prevents cleartext creds in LSASS)
- SMBv1 explicitly disabled

Clean ISO sources:
https://www.microsoft.com/en-us/software-download/
Microsoft Media Creation Tool (official)
```

***

## 2. Patching and Update Management

```powershell
# Check current patch status locally
wmic qfe get Caption,Description,HotFixID,InstalledOn | sort /+58
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 20

# Check for missing updates via PowerShell (requires PSWindowsUpdate module)
Install-Module PSWindowsUpdate -Force
Get-WUList          # list pending updates
Install-WindowsUpdate -AcceptAll -AutoReboot  # apply all

# Check Windows Update service status
Get-Service -Name wuauserv | Select-Object Name, Status, StartType
sc query wuauserv
```

```
Enterprise update management options:

WSUS (Windows Server Update Services)
- Internal update server - hosts all updates centrally
- Hosts approve/decline updates before they reach endpoints
- No individual host reaching out to Microsoft directly
- Reduces bandwidth, gives control over rollout timing

Group Policy path for WSUS:
Computer Configuration\Administrative Templates\Windows Components\Windows Update
  -> Specify intranet Microsoft update service location
  -> Configure Automatic Updates: 4 = Auto download and schedule install

Best practice for rollout:
1. Stage 1: Test VMs / dev environment (2-3 days)
2. Stage 2: Pilot group of 5-10 physical hosts (1 week)
3. Stage 3: Enterprise-wide push
Never push enterprise-wide on patch Tuesday immediately
```

***

## 3. Group Policy Hardening (Key Settings)

These are the GPO settings that directly counter the privilege escalation techniques covered throughout this module. 

```
Navigate: Group Policy Management Console (GPMC) -> gpmc.msc

Account Policies:
Computer Configuration\Windows Settings\Security Settings\Account Policies\Password Policy
  Minimum password length: 14+ characters
  Password complexity: Enabled
  Maximum password age: 60-90 days
  Password history: 24 passwords remembered
  Account lockout threshold: 5 invalid attempts
  Account lockout duration: 30 minutes

UAC (counters token impersonation, AlwaysInstallElevated):
Computer Configuration\Windows Settings\Security Settings\Local Policies\Security Options
  User Account Control: Run all administrators in Admin Approval Mode -> Enabled
  User Account Control: Behavior of the elevation prompt for admins -> Prompt for credentials
  User Account Control: Only elevate executables in signed/validated locations -> Enabled

AlwaysInstallElevated (must be DISABLED):
Computer Configuration\Administrative Templates\Windows Components\Windows Installer
  Always install with elevated privileges -> Disabled
User Configuration\Administrative Templates\Windows Components\Windows Installer
  Always install with elevated privileges -> Disabled

Credential protection:
Computer Configuration\Administrative Templates\System\Device Guard
  Turn On Virtualization Based Security -> Enabled
  Credential Guard Configuration -> Enabled with UEFI lock

Disable WDigest (prevents cleartext creds in LSASS - counters Mimikatz):
Computer Configuration\Administrative Templates\MS Security Guide
  WDigest Authentication -> Disabled
  HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest
  -> UseLogonCredential = 0

PowerShell logging (detects IEX, download cradles, obfuscated commands):
Computer Configuration\Administrative Templates\Windows Components\Windows PowerShell
  Turn on Script Block Logging -> Enabled
  Turn on Module Logging -> Enabled
  Turn on PowerShell Transcription -> Enabled

AppLocker / WDAC (blocks LOLBAS and unauthorised executables):
Computer Configuration\Windows Settings\Security Settings\Application Control Policies\AppLocker
  Executable Rules -> Allow signed Microsoft binaries, deny unsigned
  Script Rules     -> Whitelist approved scripts only
  DLL Rules        -> Block DLL side-loading paths

SMB hardening (counters Responder, NTLM relay, EternalBlue):
Computer Configuration\Administrative Templates\Network\Lanman Workstation
  Enable insecure guest logons -> Disabled
Computer Configuration\Windows Settings\Security Settings\Local Policies\Security Options
  Microsoft network server: Digitally sign communications (always) -> Enabled
  Microsoft network client: Digitally sign communications (always) -> Enabled
  Network security: LAN Manager authentication level -> Send NTLMv2 response only, refuse LM and NTLM

Disable SMBv1 via GPO:
Computer Configuration\Administrative Templates\MS Security Guide
  Configure SMBv1 Server -> Disabled
  Configure SMBv1 Client (DisableSMB1) -> Disabled
```

***

## 4. User and Account Management

```powershell
# Audit all local accounts - remove or disable unnecessary ones
Get-LocalUser | Select-Object Name, Enabled, LastLogon, Description | Format-Table -AutoSize

# Check who is in the local Administrators group
Get-LocalGroupMember -Group "Administrators"
# Flag any non-IT accounts - principle of least privilege

# Disable Guest account (should always be disabled)
Disable-LocalUser -Name "Guest"

# Rename default Administrator account (reduces attack surface)
Rename-LocalUser -Name "Administrator" -NewName "localadm_renamed"

# Check accounts with no password set (dangerous on shared machines)
Get-LocalUser | Where-Object {$_.PasswordRequired -eq $false}

# Enforce password policy via PowerShell (standalone hosts)
net accounts /minpwlen:14 /maxpwage:90 /minpwage:1 /uniquepw:24 /lockoutthreshold:5

# Review description fields for stored credentials
Get-LocalUser | Where-Object {$_.Description -ne $null -and $_.Description -ne ""} |
    Select-Object Name, Description
```

***

## 5. Credential Guard and Advanced Protection

Credential Guard isolates LSASS into a VBS (Virtualization Based Security) container, making Mimikatz-style credential dumping impossible even with SYSTEM access. 

```powershell
# Check if Credential Guard is running
Get-ComputerInfo -Property DeviceGuardSecurityServicesRunning
# Should show: CredentialGuard

# Check VBS is enabled
msinfo32 -> System Summary -> Virtualization-based security: Running

# Enable via GPO (preferred - see GPO section above)
# Or enable manually:
$featureValue = (Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard).EnableVirtualizationBasedSecurity
# Should be 1

# Verify LSASS is running in protected mode
Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Lsa -Name RunAsPPL
# RunAsPPL = 1 -> LSASS Protected Process Light enabled
# This prevents standard Mimikatz from attaching to LSASS
```

***

## 6. Logging and Monitoring

```powershell
# Sysmon installation (most valuable detection tool available)
# Download: https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon
.\sysmon64.exe -accepteula -i sysmonconfig.xml
# Use SwiftOnSecurity config: https://github.com/SwiftOnSecurity/sysmon-config

# Sysmon log location
# Applications and Service Logs\Microsoft\Windows\Sysmon\Operational

# Key Sysmon event IDs to monitor:
# Event ID 1  - Process creation (catches cmd.exe spawned from Word, etc.)
# Event ID 3  - Network connection (catches reverse shells, C2 beacons)
# Event ID 7  - Image loaded (DLL side-loading)
# Event ID 8  - CreateRemoteThread (process injection)
# Event ID 10 - ProcessAccess (LSASS access = credential dumping attempt)
# Event ID 11 - FileCreate (drops to disk)
# Event ID 13 - RegistryValue set (persistence via run keys)
# Event ID 22 - DNS query (C2 domain generation algorithm activity)

# Check Windows Security log size (default 20MB is too small)
wevtutil sl Security /ms:1073741824  # Set to 1GB

# Enable audit policies that catch privilege escalation attempts
auditpol /set /category:"Privilege Use" /success:enable /failure:enable
auditpol /set /category:"Process Tracking" /success:enable /failure:enable
auditpol /set /category:"Account Logon" /success:enable /failure:enable
auditpol /set /category:"Account Management" /success:enable /failure:enable

# View current audit policy
auditpol /get /category:*
```

***

## 7. Key Hardening Checks Mapped to Module Attack Techniques

| Attack Covered in Module | Hardening Control |
|---|---|
| Unquoted service paths | Audit: `wmic service get PathName` - fix any unquoted paths |
| Weak service ACLs | Run `accesschk` periodically, enforce least privilege ACLs |
| AlwaysInstallElevated | GPO: disable both HKCU and HKLM keys |
| Token impersonation / Potato | Enable Credential Guard, restrict SeImpersonatePrivilege |
| LSASS credential dumping | RunAsPPL=1, Credential Guard, disable WDigest |
| SCF/LNK hash capture | SMB signing required, disable LLMNR/NBT-NS |
| DLL hijacking | Audit writable dirs in PATH, WDAC application control |
| Scheduled task abuse | Audit C:\Scripts writable dirs, BUILTIN\Users permissions |
| LOLBAS abuse | AppLocker/WDAC, PowerShell Constrained Language Mode |
| Cleartext creds in files | Scan shares with Snaffler, enforce need-to-know ACLs |
| EternalBlue / MS17-010 | Disable SMBv1, patch KB4012212, network segmentation |
| mRemoteNG credential theft | Enforce config encryption, clean up stored credentials |

***

## 8. Compliance and Audit References

```
DISA STIGs (most prescriptive - DoD standard):
https://public.cyber.mil/stigs/downloads/
- Download STIG for your OS version
- Import into STIG Viewer (.ckl file)
- Step through each Rule ID, mark as Not a Finding / Open / Not Applicable
- Use SCAP Compliance Checker for automated baseline scan

Microsoft Security Compliance Toolkit:
https://www.microsoft.com/en-us/download/details.aspx?id=55319
- Pre-built GPO baselines for Windows 10, 11, Server 2019, 2022
- Import directly into GPMC or test with LGPO.exe
- Includes Policy Analyzer to compare baseline vs current settings

CIS Benchmarks:
https://www.cisecurity.org/cis-benchmarks
- Level 1: Minimum security, minimal operational impact
- Level 2: Higher security, may affect some functionality
- Free PDFs available for most Windows versions

Compliance frameworks that reference Windows hardening:
- PCI-DSS  -> applies to any system touching cardholder data
- HIPAA    -> applies to healthcare (ePHI) systems
- ISO 27001 -> general infosec management standard
- Cyber Essentials (UK) -> NCSC baseline, relevant to you in England

Key principle: frameworks are reference guides, not complete security programs.
Tailor controls to your organisation's actual threat model and data types.
```

***

## 9. Quick Hardening Audit Script

```powershell
# Run this on any host to get a snapshot of key hardening status
$report = @()

$report += [PSCustomObject]@{ Check = "SMBv1 Enabled"; Value = (Get-SmbServerConfiguration).EnableSMB1Protocol }
$report += [PSCustomObject]@{ Check = "WDigest UseLogonCredential"; Value = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest" -Name UseLogonCredential -EA SilentlyContinue).UseLogonCredential }
$report += [PSCustomObject]@{ Check = "LSASS RunAsPPL"; Value = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name RunAsPPL -EA SilentlyContinue).RunAsPPL }
$report += [PSCustomObject]@{ Check = "AlwaysInstallElevated (HKLM)"; Value = (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Installer" -Name AlwaysInstallElevated -EA SilentlyContinue).AlwaysInstallElevated }
$report += [PSCustomObject]@{ Check = "AlwaysInstallElevated (HKCU)"; Value = (Get-ItemProperty "HKCU:\SOFTWARE\Policies\Microsoft\Windows\Installer" -Name AlwaysInstallElevated -EA SilentlyContinue).AlwaysInstallElevated }
$report += [PSCustomObject]@{ Check = "Guest Account Enabled"; Value = (Get-LocalUser -Name "Guest").Enabled }
$report += [PSCustomObject]@{ Check = "Credential Guard Running"; Value = (Get-ComputerInfo -Property DeviceGuardSecurityServicesRunning).DeviceGuardSecurityServicesRunning }
$report += [PSCustomObject]@{ Check = "PowerShell Script Block Logging"; Value = (Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" -Name EnableScriptBlockLogging -EA SilentlyContinue).EnableScriptBlockLogging }

# Desired values:
# SMBv1 Enabled               -> False
# WDigest UseLogonCredential  -> 0
# LSASS RunAsPPL              -> 1
# AlwaysInstallElevated       -> 0 (or null = not set = safe)
# Guest Account Enabled       -> False
# Credential Guard Running    -> CredentialGuard
# ScriptBlockLogging          -> 1

$report | Format-Table -AutoSize
```

This script gives you an instant baseline check against the most common privilege escalation preconditions. Any item returning an insecure value is a direct finding that maps back to a specific exploit technique covered in this module. [github](https://github.com/0x6d69636b/windows_hardening)
