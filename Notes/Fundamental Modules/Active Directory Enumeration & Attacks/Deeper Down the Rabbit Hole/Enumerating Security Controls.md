## Enumerating Security Controls

After gaining a foothold, enumerating the security controls in place helps you understand what tools will work, what will get caught, and where you need to adapt your approach. Security posture varies across hosts within the same domain, so a control blocking a tool on one machine may not apply to another.

Note: This section covers identifying security controls. Bypassing them is outside the scope of this module.

***

## Windows Defender

[Microsoft Defender](https://en.wikipedia.org/wiki/Microsoft_Defender) has improved significantly over the years and will block tools like [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) by default. Use the built-in [Get-MpComputerStatus](https://docs.microsoft.com/en-us/powershell/module/defender/get-mpcomputerstatus?view=win10-ps) cmdlet to check the current Defender state:

```powershell
Get-MpComputerStatus
```

The key field to check in the output is `RealTimeProtectionEnabled`. A value of `True` confirms Defender is actively running:

```
AMServiceEnabled          : True
AntivirusEnabled          : True
BehaviorMonitorEnabled    : False
RealTimeProtectionEnabled : True
NISEnabled                : False
OnAccessProtectionEnabled : False
```

***

## AppLocker

[AppLocker](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/applocker/what-is-applocker) is Microsoft's application whitelisting solution. It gives administrators granular control over which executables, scripts, DLLs, Windows Installer files, and packaged apps can run on a system. Organisations commonly block `cmd.exe` and `PowerShell.exe` and restrict write access to certain directories.

A common oversight is blocking the standard 64-bit PowerShell path while leaving alternative locations open. Known [PowerShell executable locations](https://www.powershelladmin.com/wiki/PowerShell_Executables_File_System_Locations) that are frequently missed include:

```
%SystemRoot%\SysWOW64\WindowsPowerShell\v1.0\powershell.exe
%SystemRoot%\SysWOW64\WindowsPowerShell\v1.0\powershell_ise.exe
```

Use `Get-AppLockerPolicy` to read the effective policy and identify what is blocked and what is allowed:

```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

Example output showing a rule blocking Domain Users from running the standard 64-bit PowerShell binary:

```
PathConditions : {%SYSTEM32%\WINDOWSPOWERSHELL\V1.0\POWERSHELL.EXE}
Id             : 3d57af4a-6cf8-4e5b-acfc-c2c2956061fa
Name           : Block PowerShell
Description    : Blocks Domain Users from using PowerShell on workstations
UserOrGroupSid : S-1-5-21-2974783224-3764228556-2640795941-513
Action         : Deny

PathConditions : {%PROGRAMFILES%\*}
Name           : (Default Rule) All files located in the Program Files folder
UserOrGroupSid : S-1-1-0
Action         : Allow

PathConditions : {%WINDIR%\*}
Name           : (Default Rule) All files located in the Windows folder
UserOrGroupSid : S-1-1-0
Action         : Allow

PathConditions : {*}
Name           : (Default Rule) All files
UserOrGroupSid : S-1-5-32-544
Action         : Allow
```

In this example, only the standard 64-bit PowerShell path is blocked. Calling PowerShell from an alternative path would still work.

***

## PowerShell Constrained Language Mode

[Constrained Language Mode (CLM)](https://devblogs.microsoft.com/powershell/powershell-constrained-language-mode/) restricts PowerShell to a limited feature set. It blocks COM objects, restricts .NET types to an approved list, and prevents the use of XAML-based workflows and PowerShell classes. It is often deployed alongside AppLocker rules.

Check the current language mode with:

```powershell
$ExecutionContext.SessionState.LanguageMode
```

Possible return values:

- `FullLanguage` means no restrictions are in place
- `ConstrainedLanguage` means CLM is active and several PowerShell capabilities are unavailable

***

## LAPS

[Microsoft Local Administrator Password Solution (LAPS)](https://www.microsoft.com/en-us/download/details.aspx?id=46899) randomises and rotates local administrator passwords on domain-joined hosts, preventing lateral movement via credential reuse. When LAPS is deployed, it is worth identifying which accounts and groups can read those passwords, as that access can be leveraged during an assessment.

[LAPSToolkit](https://github.com/leoloobeek/LAPSToolkit) is a PowerShell tool that leverages [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) to audit LAPS environments. It provides three key functions:

### Find-LAPSDelegatedGroups

Identifies which groups have been explicitly delegated LAPS password read access across each OU:

```powershell
Find-LAPSDelegatedGroups
```

Example output:

```
OrgUnit                                              Delegated Groups
-------                                              ----------------
OU=Servers,DC=INLANEFREIGHT,DC=LOCAL                 INLANEFREIGHT\Domain Admins
OU=Servers,DC=INLANEFREIGHT,DC=LOCAL                 INLANEFREIGHT\LAPS Admins
OU=Workstations,DC=INLANEFREIGHT,DC=LOCAL            INLANEFREIGHT\Domain Admins
OU=Workstations,DC=INLANEFREIGHT,DC=LOCAL            INLANEFREIGHT\LAPS Admins
```

### Find-AdmPwdExtendedRights

Checks for any users or groups holding "All Extended Rights" on LAPS-enabled computers. These accounts can read LAPS passwords and may be less carefully monitored than members of explicitly delegated groups:

```powershell
Find-AdmPwdExtendedRights
```

Example output:

```
ComputerName                 Identity                     Reason
------------                 --------                     ------
EXCHG01.INLANEFREIGHT.LOCAL  INLANEFREIGHT\Domain Admins  Delegated
EXCHG01.INLANEFREIGHT.LOCAL  INLANEFREIGHT\LAPS Admins    Delegated
SQL01.INLANEFREIGHT.LOCAL    INLANEFREIGHT\Domain Admins  Delegated
WS01.INLANEFREIGHT.LOCAL     INLANEFREIGHT\Domain Admins  Delegated
```

### Get-LAPSComputers

Lists all computers with LAPS enabled, their password expiration dates, and the cleartext password if your current account has permission to read it:

```powershell
Get-LAPSComputers
```

Example output:

```
ComputerName                 Password        Expiration
------------                 --------        ----------
DC01.INLANEFREIGHT.LOCAL     6DZ[+A/[]19d$F  08/26/2020 23:29:45
EXCHG01.INLANEFREIGHT.LOCAL  oj+2A+[hHMMtj,  09/26/2020 00:51:30
SQL01.INLANEFREIGHT.LOCAL    9G#f;p41dcAe,s  09/26/2020 00:30:09
WS01.INLANEFREIGHT.LOCAL     TCaG-F)3No;l8C  09/26/2020 00:46:04
```

Recovering cleartext LAPS passwords from this output immediately provides local administrator access to the listed machines.
