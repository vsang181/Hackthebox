# Latest SQL Vulnerabilities

This section focuses on a technique rather than a traditional CVE -- there is no patch to look for and no specific version to target. The `xp_dirtree` attack works against any MSSQL instance where the service account has network access, because it abuses a legitimate Windows authentication behaviour rather than a flaw in the SQL Server software itself.

## Why This Is Not a Traditional Vulnerability

`xp_dirtree` is an undocumented MSSQL stored procedure designed to list the contents of a folder, including remote network paths. The vulnerability is not in the function itself but in what happens when it tries to access a remote share. Windows hosts automatically send an NTLMv2 hash when authenticating to a network resource -- and the MSSQL service account is no exception. By pointing `xp_dirtree` at an attacker-controlled SMB server, the authentication attempt is captured and the service account's NTLMv2 hash is surrendered without any exploit code being executed.

## What Can Be Done with the Hash

Once the NTLMv2 hash is captured, two paths are available:

- **Offline cracking** -- using hashcat with module 5600 against a wordlist. A successful crack reveals the service account's plaintext password, which may be reused across other systems or services in the environment
- **SMB Relay** -- forwarding the captured hash in real time to another host where that account holds local administrator rights. Microsoft patched the ability to relay a hash back to the originating host, but the hash can still be relayed to a different machine in the network. From there, additional credentials may be harvested and potentially used to circle back to the original host

## Mapping to the Concept of Attacks

The attack runs through two cycles.

### Cycle 1 -- Initiating the xp_dirtree Call

| Step | What Happens | Category |
|---|---|---|
| 1 | The attacker supplies a UNC path pointing to a controlled SMB server as the folder argument to `xp_dirtree` | Source |
| 2 | The MSSQL server processes the function call and attempts to list the contents of the specified remote path | Process |
| 3 | The MSSQL service executes the command under elevated service account privileges | Privileges |
| 4 | The SMB service on the attacker's machine is the destination receiving the outbound connection | Destination |

### Cycle 2 -- Capturing the Hash

| Step | What Happens | Category |
|---|---|---|
| 5 | The SMB service receives the inbound connection request triggered by the `xp_dirtree` call | Source |
| 6 | The SMB server processes the request and queries for the folder contents | Process |
| 7 | The MSSQL service account's NTLMv2 hash is used as the authentication credential in the SMB negotiation | Privileges |
| 8 | The attacker's controlled host and SMB listener is the destination -- Responder, Wireshark, or TCPDump intercepts and records the hash | Destination |

## The Broader Point

What makes this attack particularly relevant is that it requires nothing beyond a valid SQL session and network connectivity. It does not rely on a software bug, a specific version, or an unpatched CVE. The behaviour being exploited -- Windows automatically authenticating to any SMB resource a process attempts to access -- is a design characteristic, not a defect.

This also reinforces the principle covered throughout this module: database servers are not just data stores. With the right queries and sufficient privileges, they become a pivot point for credential harvesting and lateral movement across the network. As noted in the module material, there are additional execution methods in MSSQL worth exploring -- including executing Python code directly through SQL queries, covered in [Microsoft's documentation](https://docs.microsoft.com/en-us/sql/machine-learning/tutorials/quickstart-python-create-script?view=sql-server-ver15) and discussed further in a dedicated module.
