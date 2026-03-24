# Kerberos Double Hop Problem

The double hop problem occurs when you authenticate to one remote host via WinRM and then try to reach a second resource (like a Domain Controller) from that session. Unlike NTLM-based auth where the password hash is cached in memory and can be forwarded, Kerberos tickets are resource-specific. The TGS (Ticket Granting Service) ticket grants access to the immediate target, but the TGT (Ticket Granting Ticket) needed to request access to further resources is never forwarded to the remote session.

***

## Why it Happens

Think of it this way: the TGT is the master pass that lets you request tickets for new resources. The TGS is a single-use ticket for one specific service. When you connect via WinRM, only the TGS travels to the remote host. The TGT stays on your originating machine. So from the remote shell, there is no TGT to present when you try to talk to a DC or file share, and the request fails.

You can confirm this by running `klist` inside a WinRM session:

```
Cached Tickets: (1)

#0>  Client: backupadm @ INLANEFREIGHT.LOCAL
     Server: HTTP/ACADEMY-AEN-DEV01.INLANEFREIGHT.LOCAL @ INLANEFREIGHT.LOCAL
     Cache Flags: 0x8 -> ASC
```

Only a single TGS for the HTTP service is cached. There is no TGT. Compare this to an RDP session where `klist` shows four tickets including the TGT for `krbtgt/INLANEFREIGHT.LOCAL` with `Cache Flags: 0x1 -> PRIMARY` and forwardable flags, because the password is stored in memory and can be used to request new tickets at any time.

Even though the process `wsmprovhost.exe` runs under `backupadm`'s context on the remote host, the credentials themselves are not present in memory. Mimikatz confirms this: every `kerberos` section for `backupadm` shows `Password: (null)`.

***

## Workaround 1: PSCredential Object (evil-winrm)

The quickest fix when working from an evil-winrm session is to create a `PSCredential` object explicitly and pass it to every AD query using the `-Credential` flag. This re-authenticates with the DC for each request.

First, confirm the problem exists:

```powershell
get-domainuser -spn
# Exception: "An operations error occurred."
```

Then set up the credential object:

```powershell
$SecPassword = ConvertTo-SecureString '!qazXSW@' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\backupadm', $SecPassword)
```

Now pass it explicitly with every PowerView command:

```powershell
get-domainuser -spn -credential $Cred | select samaccountname
```

Output:

```
samaccountname
--------------
azureconnect
backupjob
krbtgt
mssqlsvc
sqltest
sqlprod
```

This works because the credential object forces a fresh authentication against the DC each time. The downside is that every single command needs the `-credential $Cred` flag, which becomes cumbersome with tools that do not support passing a credential object per call.

***

## Workaround 2: Register PSSession Configuration (Windows Only)

This method is cleaner and more persistent. It works by registering a named session configuration that runs as the target user, causing your local machine to impersonate the remote machine under that user's context. All subsequent requests go directly to the DC without re-specifying credentials.

### Setup (must be done from a proper PowerShell console, not evil-winrm)

Register the session on your Windows attack host or jump host:

```powershell
Register-PSSessionConfiguration -Name backupadmsess -RunAsCredential inlanefreight\backupadm
```

Restart WinRM to apply it:

```powershell
Restart-Service WinRM
```

Connect using the named configuration:

```powershell
Enter-PSSession -ComputerName DEV01 -Credential INLANEFREIGHT\backupadm -ConfigurationName backupadmsess
```

Check `klist` in the new session:

```
Cached Tickets: (1)

#0>  Client: backupadm @ INLANEFREIGHT.LOCAL
     Server: krbtgt/INLANEFREIGHT.LOCAL @ INLANEFREIGHT.LOCAL
     Cache Flags: 0x1 -> PRIMARY
     Kdc Called: DC01
```

The TGT is now cached. PowerView works without any `-Credential` flag:

```powershell
get-domainuser -spn | select samaccountname
```

***

## Method Comparison

| Factor | PSCredential Object | Register PSSession Config |
|--------|--------------------|-----------------------------|
| Works from evil-winrm | Yes | No (requires GUI/elevated PS) |
| Works from Linux attack host | Yes | No |
| Requires credentials per-command | Yes | No, set once |
| Leaves WinRM config change on target | No | Yes (needs cleanup) |
| Requires Windows attack host or RDP | No | Yes |
| TGT cached in remote session | No | Yes |

***

## Other Considerations

If **unconstrained delegation** is enabled on the host you land on, the double hop problem does not apply at all. In that configuration the TGT is sent along with the TGS in every authentication request and gets cached on the server, meaning the server can request tickets on behalf of the user to any other resource. If you land on such a box, credential access across the domain is already wide open.

Other workarounds not covered in depth here include:

- **CredSSP** (Credential Security Support Provider): delegates credentials but is considered a security risk as it exposes credentials to the remote machine
- **Port forwarding**: tunnel traffic from the remote host back through your attack machine
- **Injecting into a process** running in the context of a target user (sacrificial process technique)

For most engagements working from a Linux attack host, the PSCredential object approach is the go-to. For Windows-based assessments using RDP jump hosts, registering a PSSession configuration is cleaner and removes the friction of per-command credential passing.
