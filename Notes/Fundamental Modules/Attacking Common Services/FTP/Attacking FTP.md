# Attacking FTP

The [File Transfer Protocol (FTP)](https://en.wikipedia.org/wiki/File_Transfer_Protocol) operates on TCP port 21 and is one of the most consistently misconfigured services encountered during assessments. Its attack surface spans three main areas: anonymous access misconfigurations, brute-force credential attacks, and protocol-level abuse through the FTP bounce technique.

## Enumeration

Nmap's default script scan (`-sC`) automatically runs the [ftp-anon](https://nmap.org/nsedoc/scripts/ftp-anon.html) script, which checks whether the server permits anonymous login. The `-sV` flag pulls the service banner, which often includes the FTP daemon name and version number -- useful for identifying known CVEs or confirming the software stack before proceeding.

```bash
sudo nmap -sC -sV -p 21 192.168.2.142
```

Key things to note in the Nmap output:
- `ftp-anon: Anonymous FTP login allowed` -- anonymous access is enabled
- Directory listings show permissions, which reveal writable directories
- `[NSE: writeable]` flags directories where file uploads may be possible
- The service banner line (e.g. `220 (vsFTPd 2.3.4)`) reveals the daemon version

## Anonymous Authentication

Anonymous FTP login uses the username `anonymous` with either a blank password or any string as the password. When this is combined with misconfigured write permissions, it becomes significantly more dangerous than read-only exposure.

```bash
ftp 192.168.2.142
# Username: anonymous
# Password: (blank or any value)
```

Once inside, standard navigation and transfer commands apply:

```
ls              # list directory contents
cd <directory>  # change directory
get <file>      # download a single file
mget *          # download multiple files
put <file>      # upload a single file
mput *          # upload multiple files
help            # show available commands
```

Read access exposes whatever sensitive files have been stored on the server -- configuration files, scripts with hardcoded credentials, and internal documents. Write access opens additional attack paths. If a web application is running on the same server and has a path traversal vulnerability, an uploaded PHP file in a web-accessible FTP directory could lead directly to remote code execution.

## Brute Forcing

When anonymous access is not available, credential brute-forcing is the next step. [Medusa](https://github.com/jmk-foofus/medusa) is a reliable multi-protocol brute-force tool that handles FTP well:

```bash
medusa -u fiona -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ftp
```

The key flags break down as follows:

| Flag | Purpose |
|---|---|
| `-u` | Single username to target |
| `-U` | File containing a list of usernames |
| `-P` | File containing a list of passwords |
| `-h` | Target hostname or IP address |
| `-M` | Module to use (ftp in this case) |

A successful result is marked as `ACCOUNT FOUND` with the working credentials listed alongside it.

One important caveat: most modern applications implement account lockout policies or rate limiting that make full wordlist brute-forcing impractical and noisy. Password spraying -- testing a small number of likely passwords against many accounts -- is generally a more effective and less detectable approach during real assessments.

## FTP Bounce Attack

The FTP bounce attack abuses the `PORT` command in the FTP protocol, which was originally designed to allow a client to instruct an FTP server to send data to a third-party host. An attacker can exploit this to use an exposed FTP server as a proxy for port scanning internal hosts that are not directly accessible from the attacker's position.

The typical scenario involves two targets:
- An internet-facing FTP server (`FTP_DMZ`) that the attacker can reach directly
- An internal host (`Internal_DMZ`) that is firewalled from the internet but reachable by the FTP server

By instructing `FTP_DMZ` to connect to specific ports on `Internal_DMZ`, the attacker can determine which ports are open on the internal host without ever having a direct network path to it.

Nmap implements this via the `-b` flag:

```bash
nmap -Pn -v -n -p80 -b anonymous:password@10.10.110.213 172.17.0.2
```

The argument to `-b` takes the format `username:password@ftp-server`. Nmap connects to the FTP server, authenticates, and uses the `PORT` command to instruct it to probe the target. A successful result returns open port information for the internal host routed through the FTP server.

Modern FTP servers include protection against this by default -- they validate that the `PORT` command's destination address matches the client's IP address, preventing third-party redirects. However, older or manually misconfigured deployments remain vulnerable, and it is still worth testing during assessments against legacy infrastructure.
