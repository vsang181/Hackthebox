# FTP

The File Transfer Protocol (FTP) is one of the oldest protocols still widely encountered during network assessments. FTP operates at the **application layer** of the TCP/IP stack, alongside protocols such as HTTP and POP3. While many modern workflows have shifted to HTTPS-based file exchange, SFTP, and cloud storage, FTP remains common in legacy environments, internal networks, and quick-and-dirty administrative setups.

FTP is designed for transferring files between a client and a server, either by uploading local files to a remote host or downloading remote files to a local system. In classic FTP, communication is split into **two separate channels**:

* **Control channel (TCP/21)**
  Used for authentication, commands, and status messages.
* **Data channel (TCP/20 or ephemeral ports depending on mode)**
  Used strictly for transferring directory listings and file contents.

The control channel carries commands (for example `USER`, `PASS`, `LIST`, `RETR`, `STOR`) and the server replies with **FTP status codes** (for example `220` for service ready, `230` for login successful, `530` for authentication failure). If the data transfer is interrupted, clients can often resume transfers using features such as `REST` (restart) depending on server configuration and client support.

> FTP is typically **clear-text**. Credentials and data can be sniffed on the network if an attacker has a suitable vantage point (for example, on the same LAN segment, via ARP spoofing, or through compromised routing). Encrypted variants exist (FTPS), but many deployments still run in plain text.

---

## Active vs Passive FTP

A key concept in FTP is the difference between **active** and **passive** mode, which mainly affects how the **data channel** is established.

### Active Mode

1. Client connects to the server on **TCP/21** (control channel).
2. Client tells the server which client-side port to connect back to for the data channel (`PORT` command).
3. Server initiates the data connection **from its TCP/20** to the client-provided port.

Active mode frequently breaks when the **client is behind a firewall or NAT**, because the server’s inbound connection attempt is blocked.

### Passive Mode

1. Client connects to the server on **TCP/21** (control channel).
2. Client requests passive mode (`PASV` command).
3. Server replies with an IP/port combination.
4. Client initiates the data connection to the server’s announced port.

Passive mode works better in real networks because the **client initiates both connections**, which is generally allowed by client-side firewalls and NAT devices.

---

## Commands and Status Codes

FTP supports many commands, but implementations vary. Server-side configuration can enable or disable subsets of functionality. Typical command categories include:

* Authentication (`USER`, `PASS`)
* Navigation (`PWD`, `CWD`, `CDUP`)
* Listing (`LIST`, `NLST`)
* Transfers (`RETR` download, `STOR` upload)
* File operations (`DELE`, `RNFR`/`RNTO`, `MKD`, `RMD`)

The server responds to each command with numeric **status codes** (e.g., `200`, `220`, `226`, `230`, `530`), which can reveal whether a feature is enabled and how the server is configured.

---

## Anonymous FTP

Most FTP servers require credentials, but some are configured for **anonymous access**. This allows users to log in with `anonymous` (and typically any password, often an email address by convention). Anonymous FTP is sometimes used internally for file sharing, but when exposed externally it becomes a frequent misconfiguration.

Anonymous access is usually restricted to:

* Read-only file downloads, or
* Limited uploads to a specific directory (dropbox style)

If anonymous uploads are enabled, the risk increases significantly (data exfiltration, malicious file planting, web shell placement if the FTP directory maps to a web root, or indirect exploitation via log poisoning and parser vulnerabilities).

---

# TFTP

Trivial File Transfer Protocol (TFTP) is a simplified file transfer protocol commonly used in tightly controlled environments (for example, bootstrapping devices via PXE, network equipment firmware/config distribution). TFTP differs from FTP in several important ways:

* Uses **UDP** (default port **69**) instead of TCP
* Has **no authentication**
* Provides minimal features (no directory listing, limited command set)
* Relies on application-layer retransmissions and timeouts because UDP is unreliable

Because it has no login mechanism, access control is typically enforced by:

* OS-level file permissions on the TFTP root directory
* Server configuration restricting which directories/files are exposed
* Network segmentation (TFTP should generally be limited to local/trusted networks)

### Common TFTP Commands

| Command   | Description                                                     |
| --------- | --------------------------------------------------------------- |
| `connect` | Sets the remote host (and optionally the port) for transfers    |
| `get`     | Downloads a file (or files) from the remote host                |
| `put`     | Uploads a file (or files) to the remote host                    |
| `quit`    | Exits the client                                                |
| `status`  | Shows current settings (mode, connection state, timeouts, etc.) |
| `verbose` | Toggles verbose output                                          |

Unlike FTP, TFTP typically does **not** support directory listing, which makes blind file guessing and path knowledge more important during enumeration.

---

# Default Configuration (vsFTPd)

One of the most common FTP servers on Linux is **vsFTPd** (“Very Secure FTP Daemon”). Its default configuration is typically located at:

* `/etc/vsftpd.conf`

It is worth installing vsFTPd in a lab VM and reviewing how each setting affects behaviour, since real-world misconfigurations often come from misunderstood defaults or “temporary” changes that became permanent.

## Install vsFTPd

```bash
sudo apt install vsftpd
```

The configuration file usually contains many commented options. Not every supported setting appears in the default config file, so the full list and meaning of options should be checked via the man page.

## Reviewing vsFTPd Configuration

To view active (non-comment) settings:

```bash
cat /etc/vsftpd.conf | grep -v "#"
```

Example relevant settings:

| Setting                                   | Description                                  |
| ----------------------------------------- | -------------------------------------------- |
| `listen=NO`                               | Run from inetd or as a standalone daemon     |
| `listen_ipv6=YES`                         | Listen on IPv6                               |
| `anonymous_enable=NO`                     | Enable anonymous access                      |
| `local_enable=YES`                        | Allow local users to log in                  |
| `dirmessage_enable=YES`                   | Show directory messages                      |
| `use_localtime=YES`                       | Use local time                               |
| `xferlog_enable=YES`                      | Enable upload/download logging               |
| `connect_from_port_20=YES`                | Use port 20 for active mode data connections |
| `secure_chroot_dir=/var/run/vsftpd/empty` | Empty directory used for chroot safety       |
| `pam_service_name=vsftpd`                 | PAM service name                             |
| `rsa_cert_file=...`                       | TLS certificate path (for FTPS)              |
| `rsa_private_key_file=...`                | TLS private key path                         |
| `ssl_enable=NO`                           | Enable TLS/SSL                               |

### `/etc/ftpusers`

vsFTPd also references `/etc/ftpusers`, which is used to **deny FTP login** for specified local users, even if they exist on the system.

Example:

```bash
cat /etc/ftpusers
```

```
guest
john
kevin
```

---

# Dangerous Settings and Misconfigurations

FTP server hardening is largely configuration-driven. Some settings exist for legitimate operational use but introduce serious risk when exposed beyond a trusted network.

## Anonymous Access Options (vsFTPd)

If anonymous access is enabled, additional settings control what the anonymous user can do:

| Setting                        | Description                                                     |
| ------------------------------ | --------------------------------------------------------------- |
| `anonymous_enable=YES`         | Allow anonymous login                                           |
| `anon_upload_enable=YES`       | Allow anonymous uploads                                         |
| `anon_mkdir_write_enable=YES`  | Allow anonymous directory creation                              |
| `no_anon_password=YES`         | Do not prompt anonymous users for a password                    |
| `anon_root=/home/username/ftp` | Anonymous root directory                                        |
| `write_enable=YES`             | Enable write operations (STOR, DELE, RNFR/RNTO, MKD, RMD, etc.) |

Anonymous read access can already leak sensitive information (documents, credentials in notes, internal naming). Anonymous **write** access can be far worse, especially if:

* The FTP root maps to a web directory (file upload → web shell)
* Uploaded files are processed by automated jobs (parsers, ETL, cron tasks)
* Logs can be influenced and later included elsewhere (log injection scenarios)

---

# Anonymous Login and Basic Interaction

When connecting to an FTP server, the banner often reveals useful information (software name, version, OS hints, organisation-specific text).

```bash
ftp 10.129.14.136
```

Example interaction:

```
Connected to 10.129.14.136.
220 "Welcome to the HTB Academy vsFTP service."
Name (10.129.14.136:cry0l1t3): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```

Directory listing:

```
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-rw-rw-r--    1 1002     1002      8138592 Sep 14 16:54 Calender.pptx
drwxrwxr-x    2 1002     1002         4096 Sep 14 16:50 Clients
drwxrwxr-x    2 1002     1002         4096 Sep 14 16:50 Documents
drwxrwxr-x    2 1002     1002         4096 Sep 14 16:50 Employees
-rw-rw-r--    1 1002     1002           41 Sep 14 16:45 Important Notes.txt
226 Directory send OK.
```

To quickly view session state and mode details:

```
ftp> status
```

Useful commands for additional detail include `debug` and `trace`, which show client-side processing and raw protocol exchange:

```
ftp> debug
ftp> trace
```

---

# Banner and Metadata Hardening

Some FTP servers are configured to hide usernames and group names from directory listings:

* `hide_ids=YES`

If enabled, listings show `ftp ftp` rather than real UID/GID mappings, reducing information leakage (e.g., preventing easy local username enumeration).

Example effect:

```
-rw-rw-r--    1 ftp     ftp      8138592 Sep 14 16:54 Calender.pptx
...
-rw-------    1 ftp     ftp            0 Sep 15 14:57 testupload.txt
```

This can slow brute-force targeting by reducing leaked usernames, but it does not remove the underlying authentication risk. In practice, rate limiting and lockout tooling (e.g., fail2ban) is often used to mitigate brute-force attempts, although misconfigurations still occur.

---

# Recursive Listings

Some servers enable recursive listing for convenience:

* `ls_recurse_enable=YES`

This can be highly useful for attackers because it exposes the entire directory structure quickly:

```
ftp> ls -R
```

Recursive listings often reveal:

* Project names and customer names
* Document templates
* Backup files
* Employee folders
* File naming conventions useful for further attacks

---

# Downloading and Uploading Files

## Download a File

```
ftp> get Important\ Notes.txt
```

After download, confirm locally:

```bash
ls | grep Notes.txt
```

## Download Everything (Mirroring)

This is useful for large directory structures but can be noisy and may trigger alerting due to unusual volume.

```bash
wget -m --no-passive ftp://anonymous:anonymous@10.129.14.136
```

`wget` will create a folder named after the target IP and store all retrieved content there.

## Upload a File

If uploads are permitted:

```bash
touch testupload.txt
```

Then upload:

```
ftp> put testupload.txt
```

Upload capability is especially dangerous when the FTP content is synchronised with web server directories or consumed by automated processes. In such cases, file upload may become a path to code execution or lateral movement, depending on how the organisation uses FTP internally.

---

# Footprinting FTP with Nmap and NSE

Footprinting services with scanners is a common approach, especially when services run on non-standard ports. Nmap includes the **Nmap Scripting Engine (NSE)**, which provides service-specific checks.

Update the NSE database:

```bash
sudo nmap --script-updatedb
```

Locate FTP-related scripts:

```bash
find / -type f -name ftp* 2>/dev/null | grep scripts
```

Example FTP scripts include:

* `ftp-anon.nse` (checks anonymous login)
* `ftp-syst.nse` (queries system/status info)
* `ftp-brute.nse` (brute force testing)
* Vulnerability checks for specific historical issues and backdoors

Basic scan against standard FTP port:

```bash
sudo nmap -sV -p21 -sC -A 10.129.14.136
```

Example findings may include:

* Service fingerprint (e.g., vsFTPd)
* Anonymous login availability
* Directory listing if anonymous is enabled
* Server status output via `STAT`

To see network-level script interaction:

```bash
sudo nmap -sV -p21 -sC -A 10.129.14.136 --script-trace
```

This can help confirm what commands are sent (e.g., `SYST`, `STAT`, `PASV`, `LIST`) and what responses are returned.

---

# Manual Service Interaction

For basic banner grabbing and manual interaction:

```bash
nc -nv 10.129.14.136 21
```

or:

```bash
telnet 10.129.14.136 21
```

If the FTP server supports TLS (FTPS), use OpenSSL to negotiate encryption and inspect the certificate:

```bash
openssl s_client -connect 10.129.14.136:21 -starttls ftp
```

Certificates can reveal:

* Hostnames (CN/SAN values)
* Organisation/unit fields
* Email addresses in certificate subject fields (occasionally present)
* Geographic or site-specific metadata (sometimes used in multi-region environments)

This information can feed back into domain enumeration and OSINT mapping, especially when certificate subjects reference internal naming conventions such as `master.<domain>` or administrative contact addresses.
