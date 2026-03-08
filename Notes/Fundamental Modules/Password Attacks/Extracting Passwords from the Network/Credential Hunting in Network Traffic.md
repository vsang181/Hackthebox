# Credential Hunting in Network Traffic

Credential hunting in network traffic exploits the fact that many real-world environments still run legacy or misconfigured services that transmit sensitive data without encryption. Even in otherwise hardened networks, a single misconfigured service, an old internal tool, or an unencrypted management protocol can expose credentials directly on the wire.

## Cleartext vs Encrypted Protocols

Understanding which protocols expose credentials in plaintext is the foundation of this technique. The following table maps common unencrypted protocols to their secure counterparts:

| Unencrypted Protocol | Encrypted Counterpart | What Gets Exposed |
|---|---|---|
| HTTP | HTTPS | Form submissions, Basic Auth headers, cookies |
| FTP | FTPS / SFTP | Username and password in AUTH commands |
| SNMP (v1/v2) | SNMPv3 with encryption | Community strings (function as passwords) |
| POP3 | POP3S | Email account credentials |
| IMAP | IMAPS | Email account credentials |
| SMTP | SMTPS | Email sending credentials |
| LDAP | LDAPS | Bind credentials (usernames and passwords) |
| RDP | RDP with TLS | Credentials in older negotiation-mode configurations |
| DNS | DNS over HTTPS (DoH) | Internal hostnames and query patterns |
| SMB (v1/early v2) | SMB over TLS (SMB 3.0) | NTLM challenge-response hashes |
| VNC | VNC with TLS/SSL | Authentication handshake material |

In modern environments, the most likely sources of cleartext traffic are internal tooling, legacy administrative interfaces, SNMP-managed network devices, and old FTP-based file transfer processes that were never migrated to SFTP.

## Wireshark

[Wireshark](https://www.wireshark.org/) is the standard tool for interactive packet analysis and comes pre-installed in most penetration testing distributions. Its [display filter engine](https://www.wireshark.org/docs/man-pages/wireshark-filter.html) allows precise narrowing of packet captures to the traffic most likely to contain credentials.

### Useful Display Filters

| Filter | Purpose |
|---|---|
| `http` | Show all HTTP traffic |
| `http.request.method == "POST"` | Isolate POST requests — form logins, API calls with credentials |
| `http contains "passw"` | Find packets where the payload contains password-related strings |
| `ftp` | Show all FTP control channel traffic (credentials appear here) |
| `smtp` | Show SMTP traffic — AUTH commands expose credentials in Base64 |
| `ldap` | Show LDAP bind requests — credentials visible in simple binds |
| `snmp` | Show SNMP traffic — community strings visible in all v1/v2 packets |
| `tcp.port == 21` | Filter FTP specifically by port |
| `ip.src == 192.168.1.5 && ip.dst == 192.168.1.10` | Track traffic between two specific hosts |
| `tcp.stream eq 53` | Follow a single TCP conversation end-to-end |
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | Identify SYN-only packets — useful for spotting scanning activity |

### Following TCP Streams

For protocols like FTP, SMTP, and POP3 where credentials are exchanged across multiple packets, **Follow TCP Stream** (`Right-click → Follow → TCP Stream`) reassembles the full conversation into readable text. This is the fastest way to see an FTP session with `USER` and `PASS` commands in sequence, or an SMTP `AUTH LOGIN` exchange with Base64-encoded credentials.

### Searching Packet Contents

To find credentials embedded in packet payloads, use either the display filter `http contains "passw"` or navigate to `Edit > Find Packet`, set the search type to `String`, and search for terms like `password`, `passwd`, `Authorization`, or `login`. This is effective for HTTP Basic Auth headers (`Authorization: Basic <base64>`) and HTML form submissions containing credential field names.

> **HTTP Basic Auth note:** The `Authorization: Basic` header value is Base64-encoded, not encrypted. Decoding it with `echo "<value>" | base64 -d` immediately reveals the `username:password` pair in plaintext.

## PCredz

[PCredz](https://github.com/lgandx/PCredz) automates credential extraction from packet capture files or live interfaces, replacing manual Wireshark analysis for bulk traffic review. It is significantly faster than manual inspection when dealing with large captures. All extracted hashes are output in [Hashcat-compatible format](https://hashcat.net/wiki/doku.php?id=example_hashes) for immediate cracking:

- NTLMv2 hashes: Hashcat mode `5600`
- NTLMv1 hashes: Hashcat mode `5500`
- Kerberos AS-REQ hashes: Hashcat mode `7500`

PCredz can be run against a saved capture file or against a live interface. Installation is via the GitHub repository or the provided Docker container detailed in the [README](https://github.com/lgandx/PCredz?tab=readme-ov-file#install):

```bash
# Against a capture file
./Pcredz -f demo.pcapng -t -v

# Against a live interface
sudo ./Pcredz -i eth0 -t -v
```

The `-t` flag enables credit card number scanning and the `-v` flag increases verbosity. All extracted credentials are simultaneously written to `CredentialDump-Session.log` in the working directory.

Example output from a capture file containing SNMP and FTP traffic:

```
[1746131482.601354] protocol: udp 192.168.31.211:59022 > 192.168.31.238:161
Found SNMPv2 Community string: s3cr...

[1746131482.658938] protocol: tcp 192.168.31.243:55707 > 192.168.31.211:21
FTP User: le...
FTP Pass: qw...
```

### What PCredz Extracts

| Protocol/Type | What is Captured |
|---|---|
| HTTP Basic Auth | Decoded username and password |
| HTTP Forms | Field values submitted via POST |
| HTTP NTLM | NTLMv1/v2 challenge-response hashes |
| SMBv1/v2 | NTLMv2 hashes from authentication attempts |
| DCE-RPC / MSSQL / LDAP | NTLM hashes from protocol-level authentication |
| FTP | Cleartext username and password |
| SMTP / POP3 / IMAP | Cleartext or Base64-encoded credentials |
| SNMP v1/v2 | Community strings |
| Kerberos | AS-REQ Pre-Auth etype 23 hashes (crackable offline) |
| Credit card numbers | PAN numbers via regex matching |

## Workflow Summary

| Step | Tool | Action |
|---|---|---|
| Capture or obtain traffic | Wireshark / `tcpdump` | Live capture or load `.pcap`/`.pcapng` file |
| Quick automated sweep | `PCredz -f capture.pcapng -v` | Extract all supported credential types at once |
| Manual investigation | Wireshark display filters | Follow TCP streams, search for credential strings |
| Decode Basic Auth | `echo "<b64>" \| base64 -d` | Recover username:password from HTTP headers |
| Crack extracted hashes | `hashcat -m 5600` (NTLMv2) | Offline password recovery from captured NTLM/Kerberos |
