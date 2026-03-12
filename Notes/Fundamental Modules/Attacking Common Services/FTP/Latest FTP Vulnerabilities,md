# Latest FTP Vulnerabilities

This section applies the Concept of Attacks framework introduced earlier to a real, documented FTP vulnerability, showing how the four categories map onto an actual exploit rather than an abstract description.

## The Vulnerability

[CVE-2022-22836](https://nvd.nist.gov/vuln/detail/CVE-2022-22836) affects CoreFTP before build 727. The service is designed to handle file uploads via HTTP POST requests, but it also accepts HTTP PUT requests without applying the same path restrictions. This combination of two separate weaknesses -- a path traversal flaw and improper write permission controls -- allows an authenticated attacker to write files to any location on the underlying system, not just the directories the FTP service is authorised to access.

## The Exploit

The entire attack is delivered as a single cURL command:

```bash
curl -k -X PUT -H "Host: <IP>" --basic -u <username>:<password> \
--data-binary "PoC." --path-as-is https://<IP>/../../../../../../whoops
```

Breaking down each component:

| Flag | Purpose |
|---|---|
| `-k` | Skip TLS certificate validation |
| `-X PUT` | Use an HTTP PUT request instead of POST |
| `-H "Host: <IP>"` | Set the Host header to the target IP |
| `--basic -u <username>:<password>` | Authenticate with HTTP Basic Auth |
| `--data-binary "PoC."` | The content to be written to the file |
| `--path-as-is` | Prevent cURL from normalising the path and stripping the traversal sequences |
| `/../../../../../../whoops` | Path traversal sequence to escape the restricted directory, with the filename at the end |

The `--path-as-is` flag is critical here. Without it, cURL would resolve the `../` sequences before sending the request, collapsing the path and neutralising the traversal. Passing it raw forces the server to interpret the path itself, where the vulnerable application fails to sanitise it correctly.

## Mapping to the Concept of Attacks

The exploit runs through two complete cycles.

### Cycle 1 -- Directory Traversal

| Step | What Happens | Category |
|---|---|---|
| 1 | The attacker supplies an HTTP PUT request with `../` traversal sequences embedded in the file path | Source |
| 2 | The CoreFTP service accepts the PUT request and begins processing the path provided | Process |
| 3 | The application checks permissions against the restricted folder only -- the traversal sequences move execution outside that folder, bypassing the restriction entirely | Privileges |
| 4 | The resulting destination is a location outside the authorised directory, passed to the write process | Destination |

### Cycle 2 -- Arbitrary File Write

| Step | What Happens | Category |
|---|---|---|
| 5 | The filename (`whoops`) and file contents (`PoC.`) provided by the attacker become the input for the write operation | Source |
| 6 | The process proceeds to write the specified content to the specified file at the traversed path | Process |
| 7 | Because all path restrictions were already bypassed in cycle 1, the service has no remaining controls to block the write | Privileges |
| 8 | The file `whoops` containing `PoC.` is written to `C:\` on the target system | Destination |

The result is confirmed on the target:

```cmd
C:\> type C:\whoops
PoC.
```

## The Broader Lesson

What makes this vulnerability useful as a teaching example is how clearly it demonstrates that a single logical flaw rarely operates in isolation. The path traversal (cycle 1) would be far less impactful without the improper write permissions (cycle 2), and the write primitive would be containable if the traversal were not possible. The two weaknesses compound each other into a meaningful capability -- writing attacker-controlled content anywhere on the filesystem.

In a real engagement, this primitive extends well beyond writing a proof-of-concept string. Depending on the server configuration, writing to startup directories, web roots, or scheduled task locations could lead directly to persistence or code execution. Understanding the mechanics through the Concept of Attacks framework makes it straightforward to reason about what else becomes possible once the initial primitive is established.
