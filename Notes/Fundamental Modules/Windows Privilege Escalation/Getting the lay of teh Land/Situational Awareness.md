## Situational Awareness

Situational awareness is the mandatory first step after gaining a foothold. Every subsequent decision, which tools to use, which vectors to pursue, which credentials to hunt, depends entirely on what the current environment looks like. Rushing straight to exploitation without this context wastes time and risks burning your access. 

***

## Network Information

Network data serves two purposes simultaneously: it informs local privilege escalation by revealing service configurations and accessible resources, and it maps the path to lateral movement once you have elevated access. 
### Interface and IP Enumeration

```cmd
:: Full interface, IP, DNS, and DHCP detail
ipconfig /all

:: What to look for:
:: - Multiple Ethernet adapters = dual-homed host, potential pivot point
:: - Static IPs = likely server, more valuable
:: - DNS Suffix (e.g. .corp.local) = domain-joined, Active Directory in play
:: - DHCP server IP = likely gateway, sometimes the DC

:: Flush and display cached DNS (reveals internal hostnames recently resolved)
ipconfig /displaydns
```

Reading dual-homed output:

```
Ethernet adapter Ethernet0:
    IPv4 Address: 10.129.43.8       <- external / lab facing

Ethernet adapter Ethernet1:
    IPv4 Address: 192.168.20.56     <- internal network, invisible from outside
```

Finding `Ethernet1` connected to `192.168.20.0/24` immediately opens a pivot path to an entire internal segment you may not have been able to reach before. 
### ARP Cache

```cmd
arp -a

:: What to look for:
:: - dynamic entries = hosts this machine has actively communicated with recently
:: - Multiple /24 subnets listed = confirms dual-homed access
:: - Repeated admin IPs = where RDP/WinRM connections originate from
```

The ARP cache is particularly valuable for identifying admin workstations. If an admin regularly RDPs into this server, their machine's IP will appear as a dynamic ARP entry, and their cached credentials may be accessible after escalation. 

### Routing Table

```cmd
route print

:: What to look for:
:: - Multiple default gateways (0.0.0.0) = multi-homed routing
:: - Specific /24 or /16 routes = reveals subnets this host can reach
:: - Persistent routes = manually configured, often indicate deliberate connectivity
```

### Active Network Connections

```cmd
:: Shows all connections, listening ports, and PID of owning process
netstat -ano

:: Cross-reference PID to process name
tasklist /svc /FI "PID eq <pid>"

:: What to look for:
:: - Ports only bound to 127.0.0.1 = internal services not exposed externally
::   (databases, admin panels, internal APIs accessible after pivot)
:: - ESTABLISHED connections from admin IPs = active admin sessions
:: - LISTENING on unusual ports = potential services not in scope of initial scan
```

***

## Enumerating Protections

Understanding what defences are deployed changes everything about how you proceed. Running WinPEAS against a fully monitored EDR will generate alerts immediately. Knowing the defensive stack lets you choose lower-footprint manual checks first. 

### Windows Defender

```powershell
# Full status of Defender components
Get-MpComputerStatus

# Key fields to read:
# RealTimeProtectionEnabled : False  <- Defender is off, tools run freely
# AMServiceEnabled          : True   <- service is running
# BehaviorMonitorEnabled    : False  <- no behavioural analysis
# IoavProtectionEnabled     : False  <- no scan on download

# Quick one-liner check
Get-MpComputerStatus | Select-Object RealTimeProtectionEnabled, AMServiceEnabled, BehaviorMonitorEnabled

# Check via registry (useful if PowerShell is restricted)
reg query "HKLM\SOFTWARE\Microsoft\Windows Defender" /v DisableAntiSpyware
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v DisableAntiVirus
```

### Third-Party AV and EDR Products

```powershell
# List all running processes to spot AV/EDR agents
tasklist /svc | findstr -i "carbon\|cylance\|sentinel\|crowd\|tanium\|defender\|mcafee\|symantec\|sophos\|trend\|eset\|kaspersky"

# Check installed programs
wmic product get name | findstr -i "endpoint\|protect\|security\|antivirus\|detect"

# Check services
Get-Service | Where-Object {$_.DisplayName -match "Protect|Defend|Endpoint|Security|AV|EDR"} |
    Select-Object DisplayName, Status, StartType

# Check running services via sc
sc query type= all | findstr -i "carbon\|cylance\|crowdstrike\|sentinel"
```

Common EDR process names to look for:

```
MsMpEng.exe         Windows Defender
CSFalconService.exe CrowdStrike Falcon
SentinelAgent.exe   SentinelOne
CylanceSvc.exe      Cylance
cb.exe              Carbon Black
taniumclient.exe    Tanium
```

### AppLocker

```powershell
# View effective (currently enforced) AppLocker policy rules
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections

# View locally configured rules only
Get-AppLockerPolicy -Local | select -ExpandProperty RuleCollections

# Test whether a specific binary is allowed or denied for a user
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -User Everyone

# Check via registry (works without PowerShell module)
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\SrpV2"
```

Reading AppLocker output:

```
Default AppLocker rules (allow Everyone):
  %PROGRAMFILES%\*      <- C:\Program Files and C:\Program Files (x86)
  %WINDIR%\*            <- C:\Windows

Default rule (allow Administrators):
  *                     <- everything

If cmd.exe returns PolicyDecision: Denied, you need an AppLocker bypass.
```

### AppLocker Bypass Paths

If AppLocker blocks your binaries, look for writable subdirectories within the whitelisted paths. AppLocker allows everything under `C:\Windows\*` and `C:\Program Files\*`, so any writable subdirectory within those is a bypass.

```powershell
# Find writable directories within AppLocker-whitelisted paths
Get-ChildItem "C:\Windows" -Recurse -Directory -ErrorAction SilentlyContinue |
    ForEach-Object {
        $acl = Get-Acl $_.FullName -ErrorAction SilentlyContinue
        if ($acl.AccessToString -match "Everyone.*Allow.*Write|Users.*Allow.*Write") {
            $_.FullName
        }
    }

# Common writable whitelisted locations
C:\Windows\Temp                  <- writable by all users
C:\Windows\System32\spool\drivers\color  <- often writable
C:\Windows\Tasks                 <- writable by all users
```

### PowerShell Constrained Language Mode

When AppLocker script rules are active, PowerShell automatically enters Constrained Language Mode, which blocks many attack techniques.

```powershell
# Check current language mode
$ExecutionContext.SessionState.LanguageMode
# FullLanguage  = no restrictions
# ConstrainedLanguage = AppLocker/WDAC script rules active, many .NET calls blocked

# If constrained, downgrade to PowerShell 2 (no constrained language mode support)
powershell -version 2
# Check again after downgrade
$ExecutionContext.SessionState.LanguageMode
```

***

## Full Situational Awareness Checklist

```cmd
REM Network
ipconfig /all
arp -a
route print
netstat -ano

REM System identity
systeminfo
hostname
echo %COMPUTERNAME%
echo %USERDOMAIN%

REM Active Directory context
nltest /dsgetdc:%USERDOMAIN% 2>nul         :: Get domain controller IP
net view /domain 2>nul                     :: Other domains visible
wmic ntdomain get DomainName,DomainControllerName

REM Defenses
powershell -c "Get-MpComputerStatus | Select RealTimeProtectionEnabled,AMServiceEnabled"
powershell -c "Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections"
tasklist /svc | findstr -i "defender\|carbon\|cylance\|crowd\|sentinel\|tanium"

REM Environment variables (may reveal credentials, custom paths, or interesting configs)
set
```
