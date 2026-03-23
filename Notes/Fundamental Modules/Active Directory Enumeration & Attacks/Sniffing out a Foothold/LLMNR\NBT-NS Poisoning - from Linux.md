# LLMNR/NBT-NS Poisoning - from Linux

LLMNR and NBT-NS poisoning is one of the most reliably effective techniques for gaining an initial foothold in an internal AD environment. It requires no prior credentials, no existing access to any host, and leverages a design flaw built into Windows name resolution that has existed for decades. In many environments it produces valid domain user hashes within minutes of starting.

## Understanding the Vulnerability

When a Windows host needs to resolve a hostname, it works through a priority chain. DNS is tried first. If DNS cannot resolve the name, the host falls back to LLMNR, sending a multicast query to `224.0.0.252` on UDP port 5355 asking all hosts on the local segment if anyone knows the address. If LLMNR also fails, NBT-NS broadcasts on UDP port 137 asking the same question using the older NetBIOS name format.

The critical flaw in both protocols is that they accept a response from any host on the network without any verification of whether the responding host is authoritative or legitimate. There is no authentication of the responder. This means any machine on the same broadcast domain can intercept these broadcasts and reply, claiming to be the host being searched for. When the requesting host accepts that reply and attempts to connect, it sends its NTLM credentials to the attacker's machine.

The most common trigger in practice is a user mistyping a hostname or UNC path. A user trying to access `\\fileserver01` who types `\\fileserv01` instead will send a DNS query that fails, then fall through to LLMNR and NBT-NS broadcasts that Responder captures and responds to. The user's machine then sends an NTLMv2 authentication attempt directly to the attacker.

## Attack Flow Step by Step

The complete sequence from broadcast to hash capture:

1. A host on the network attempts to resolve `\\printer01.inlanefreight.local`
2. The DNS server has no record for this name and responds accordingly
3. The host broadcasts an LLMNR query to the entire local segment asking if anyone knows `printer01`
4. Responder, listening on the attacker's machine, intercepts the broadcast and immediately replies claiming to be `printer01`
5. The requesting host trusts the reply and initiates an SMB authentication attempt to the attacker
6. The attacker's machine captures the NTLMv2 challenge-response hash for the user initiating the connection

The hash captured is a NetNTLMv2 hash. It is not an NTLM hash and cannot be used directly for pass-the-hash attacks. It must either be cracked offline to recover the plaintext password, or relayed to another service in real time using `ntlmrelayx.py` if SMB signing is not enforced on the target. Relay attacks are covered separately in the lateral movement module.

## Running Responder

Start Responder with default settings on the network interface facing the internal segment. Running it in a tmux or screen session allows it to capture hashes in the background while other enumeration continues in parallel:

```bash
sudo responder -I ens224
```

With defaults, Responder spins up rogue SMB, HTTP, HTTPS, LDAP, FTP, MSSQL, and several other protocol listeners simultaneously. When any host on the segment authenticates to any of these rogue services, the hash is captured.

Key flags to know:

- `-A` -- analyze mode only, listens without responding, used for passive recon earlier in the assessment
- `-w` -- starts the WPAD rogue proxy server, extremely effective in large organizations where hosts using Internet Explorer with auto-detect proxy settings will send authentication when they request the `wpad.dat` file
- `-f` -- fingerprints the remote host's OS and version from the NBT-NS or LLMNR query
- `-v` -- verbose output, useful for troubleshooting but produces a significant amount of additional terminal noise
- `-F` -- forces NTLM or Basic authentication on WPAD file retrieval, may trigger a visible login prompt for the user so use carefully

The ports Responder needs available to function fully are:

```
UDP 137, UDP 138, UDP 53, UDP/TCP 389
TCP 1433, UDP 1434, TCP 80, TCP 135, TCP 139
TCP 445, TCP 21, TCP 3141, TCP 25, TCP 110
TCP 587, TCP 3128, Multicast UDP 5355 and 5353
```

Any individual rogue server can be disabled in `/usr/share/responder/Responder.conf` if there is a conflict with an existing service.

## Captured Hash Storage

All captured hashes are written to `/usr/share/responder/logs/` in two places: individual text files per host and protocol, and a centralised SQLite database. The file naming convention is:

```
(MODULE_NAME)-(HASH_TYPE)-(CLIENT_IP).txt
```

Example files from a successful capture session:

```
SMB-NTLMv2-SSP-172.16.5.25.txt
SMB-NTLMv2-SSP-172.16.5.50.txt
HTTP-NTLMv2-172.16.5.200.txt
Proxy-Auth-NTLMv2-172.16.5.200.txt
```

Each file contains the full hash string in a format ready for Hashcat. One hash per unique user per protocol is printed to the console in standard mode; verbose mode prints all attempts.

## Cracking the Hash Offline

NTLMv2 hashes use Hashcat mode `5600`. A captured hash for user `FOREND` against rockyou.txt:

```bash
hashcat -m 5600 forend_ntlmv2 /usr/share/wordlists/rockyou.txt
```

A successful crack returns the plaintext password directly:

```
FOREND::INLANEFREIGHT:4af70a79938ddf8a:0f85ad1e80baa52d...:Klmcargo2

Status: Cracked
Time.Started: Mon Feb 28 15:20:30 2022 (11 secs)
```

The password `Klmcargo2` is now a valid domain credential that can be used for authenticated enumeration of the entire INLANEFREIGHT.LOCAL domain.

NTLMv2 is slower to crack than NTLM because of the challenge-response structure of the hash. On a GPU cracking rig it is tractable for most passwords under 10 characters using common wordlists and rules. Long, complex, or randomly generated passwords may not be crackable within an assessment timeframe, which is why the relay attack path matters as a parallel option. If you cannot crack the hash, you may still be able to relay it to gain access without ever knowing the plaintext password.

If you obtain an unfamiliar hash format, the [Hashcat example hashes wiki](https://hashcat.net/wiki/doku.php?id=example_hashes) is the fastest way to identify the correct mode number.

## Operational Considerations

Several practical points affect how you run Responder on a real engagement:

Running Responder continuously in the background throughout the assessment day maximises the number of hashes captured. Morning logon activity, scheduled tasks, and users browsing file shares all generate LLMNR and NBT-NS traffic. The longer Responder runs, the more material you collect.

On an evasive engagement, Responder is loud. Every rogue response it sends appears as network traffic, and a SIEM with LLMNR poisoning detection rules will generate alerts quickly. In those cases, analyse mode first to understand traffic patterns, and then limit active poisoning to specific windows of time or specific hosts.

The `-w` WPAD flag deserves specific attention in environments with older Windows versions or where Internet Explorer is used as the default browser. When a Windows host has the "Automatically detect settings" option enabled in IE proxy settings, it will attempt to download a `wpad.dat` proxy configuration file by resolving `wpad` as a hostname. This triggers LLMNR and NBT-NS lookups that Responder intercepts, and the resulting authentication attempt often comes from a service account or the user's browser session rather than requiring any user action at all.
