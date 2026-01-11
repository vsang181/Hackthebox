# SMB

Server Message Block (SMB) is a client-server protocol used to access remote files, directories, and other shared network resources such as printers and named pipes. It is most commonly associated with Windows file sharing, but it is also widely used in mixed environments through **Samba**, which provides SMB services on Linux/Unix systems.

From a penetration testing perspective, SMB is a high-value protocol because it often exposes:

* File shares containing sensitive documents, scripts, or credentials
* User and group information via RPC
* Authentication behaviour (null sessions, guest access, signing requirements)
* Domain-related metadata (workgroup/domain names, server roles, OS hints)

---

## How SMB Works (High Level)

SMB allows a client to communicate with other systems on the same network to access shared resources. Before file operations occur, the client and server must negotiate a session and exchange protocol messages.

In IP networks, SMB uses **TCP**, so a connection begins with a standard TCP three-way handshake. Once established, the SMB session negotiates:

* SMB dialect/version (SMB1 vs SMB2/3)
* Authentication method (guest, NTLM, Kerberos in AD environments)
* Optional security controls (message signing, encryption in SMB3)

An SMB server can publish arbitrary parts of its filesystem as **shares**, meaning the structure visible to clients does not have to match the server’s local directory layout. Access control is typically implemented via **ACLs** (Access Control Lists). These can be defined per share and can differ from local filesystem permissions, depending on configuration and how Samba/Windows enforces share permissions.

---

## Samba and SMB/CIFS

**Samba** is the standard SMB implementation on Unix-like systems. You will often see “SMB/CIFS” used together because Samba historically supported the **CIFS** dialect (commonly aligned with SMB1).

Important port behaviour:

* SMB over NetBIOS typically uses **TCP/137, UDP/138, TCP/139** (legacy environments)
* Modern SMB typically uses **TCP/445** directly (preferred)

SMB versions have evolved over time. SMB1 (CIFS) is considered outdated and risky, while SMB2/SMB3 introduce security and performance improvements (including encryption in SMB3).

### SMB Versions (Quick Reference)

| SMB Version | Supported (Examples)        | Key Features                                          |
| ----------- | --------------------------- | ----------------------------------------------------- |
| CIFS / SMB1 | Windows NT 4.0 era          | NetBIOS-based communication (legacy)                  |
| SMB 1.0     | Windows 2000                | Direct TCP support                                    |
| SMB 2.0     | Windows Vista / Server 2008 | Performance improvements, better signing behaviour    |
| SMB 2.1     | Windows 7 / Server 2008 R2  | Improved locking and caching mechanisms               |
| SMB 3.0     | Windows 8 / Server 2012     | Multichannel, encryption support, improved resilience |
| SMB 3.1.1   | Windows 10 / Server 2016    | Pre-auth integrity, stronger crypto options           |

Samba can also integrate deeply with Windows environments:

* Samba v3 can join an AD domain as a member server
* Samba v4 can act as an **Active Directory Domain Controller**

Samba services are typically handled by background daemons, such as:

* `smbd` for file sharing and SMB services
* `nmbd` for NetBIOS name services (legacy)

---

## Workgroups, NetBIOS, and Naming

In SMB networks, hosts may be grouped under a **workgroup** name, which is a logical identifier for an arbitrary collection of computers and resources. In legacy NetBIOS environments, systems register names through:

* Local name registration, or
* A NetBIOS Name Server (**NBNS**), often implemented as **WINS** in Windows networks

These naming systems can leak useful metadata such as hostnames, workgroup names, and domain identifiers.

---

# Default Samba Configuration

Samba configuration is typically stored in:

* `/etc/samba/smb.conf`

To view active settings (excluding comments):

```bash
cat /etc/samba/smb.conf | grep -v "#\|\;"
```

Example configuration:

```text
[global]
   workgroup = DEV.INFREIGHT.HTB
   server string = DEVSMB
   log file = /var/log/samba/log.%m
   max log size = 1000
   logging = file
   panic action = /usr/share/samba/panic-action %d

   server role = standalone server
   obey pam restrictions = yes
   unix password sync = yes

   passwd program = /usr/bin/passwd %u
   passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .

   pam password change = yes
   map to guest = bad user
   usershare allow guests = yes

[printers]
   comment = All Printers
   browseable = no
   path = /var/spool/samba
   printable = yes
   guest ok = no
   read only = yes
   create mask = 0700

[print$]
   comment = Printer Drivers
   path = /var/lib/samba/printers
   browseable = yes
   read only = yes
   guest ok = no
```

Samba has **global settings** and **per-share settings**. Share definitions can override global behaviour. This flexibility is useful operationally, but it also increases the likelihood of misconfigurations where a single share becomes overly permissive.

### Common Samba Settings

| Setting                        | Description                               |
| ------------------------------ | ----------------------------------------- |
| `[sharename]`                  | Name of the network share                 |
| `workgroup = WORKGROUP/DOMAIN` | Workgroup/domain shown to clients         |
| `path = /path/here/`           | Directory exposed via the share           |
| `server string = STRING`       | Banner-like description shown to clients  |
| `unix password sync = yes`     | Sync UNIX password with SMB password      |
| `usershare allow guests = yes` | Allow guest access to user-defined shares |
| `map to guest = bad user`      | Map unknown users to the guest account    |
| `browseable = yes`             | Show share in share listings              |
| `guest ok = yes`               | Allow access without credentials          |
| `read only = yes`              | Restrict to read-only access              |
| `create mask = 0700`           | Default permissions for new files         |

---

# Dangerous Settings (What to Watch For)

Some Samba settings are particularly risky because they increase visibility or allow unauthenticated read/write access.

| Setting                                        | Security Impact                                              |
| ---------------------------------------------- | ------------------------------------------------------------ |
| `browseable = yes`                             | Attackers can enumerate shares more easily                   |
| `guest ok = yes`                               | Enables unauthenticated access (often a critical issue)      |
| `read only = no` / `writable = yes`            | Enables file modification and uploads                        |
| `create mask = 0777` / `directory mask = 0777` | Creates files/directories with overly permissive permissions |
| `logon script = script.sh`                     | Can be abused if attackers can modify the script or its path |
| `magic script` / `magic output`                | Dangerous if misused; can create unexpected execution paths  |

A common real-world pattern is “temporary” permissive settings added for testing or internal convenience and then forgotten, leaving a share open.

### Example Insecure Share

```text
[notes]
    comment = CheckIT
    path = /mnt/notes/

    browseable = yes
    read only = no
    writable = yes
    guest ok = yes

    enable privileges = yes
    create mask = 0777
    directory mask = 0777
```

After editing Samba configuration, restart the SMB daemon:

```bash
sudo systemctl restart smbd
```

---

# Enumerating SMB Shares with smbclient

To list shares on a target host:

* `-L` lists available shares
* `-N` attempts a null session (no password prompt)

```bash
smbclient -N -L //10.129.14.128
```

Example output:

```text
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
home            Disk      INFREIGHT Samba
dev             Disk      DEVenv
notes           Disk      CheckIT
IPC$            IPC       IPC Service (DEVSM)
SMB1 disabled -- no workgroup available
```

To connect to a specific share:

```bash
smbclient //10.129.14.128/notes
```

Once connected, `help` shows available commands. Commonly used commands include:

* `ls` / `dir` to list files
* `get <file>` to download
* `put <file>` to upload (if permitted)
* `recurse ON` and `mget *` for bulk retrieval
* `!<cmd>` to run a local shell command without leaving smbclient

Example: download a file and verify locally:

```text
smb: \> get prep-prod.txt
```

```text
smb: \> !ls
smb: \> !cat prep-prod.txt
```

---

# Server-Side Visibility (smbstatus)

From an administrator’s perspective (or during post-exploitation), `smbstatus` shows active Samba sessions and connected shares:

```bash
smbstatus
```

It can reveal:

* Samba version
* Connected users
* Source IPs
* Shares in use
* Signing/encryption status (depending on build/config)

This is useful for defenders, but from an attacker viewpoint it confirms that SMB sessions are tracked and may be monitored.

---

# Footprinting SMB with Nmap

SMB typically runs on:

* TCP/445 (modern SMB)
* TCP/139 (NetBIOS SMB, legacy)

A basic service scan:

```bash
sudo nmap 10.129.14.128 -sV -sC -p139,445
```

Common NSE output includes:

* SMB dialects supported (SMB2/3 vs SMB1)
* Whether message signing is enabled/required
* NetBIOS name information
* Server time and other metadata

Nmap is useful for quick triage, but manual enumeration usually reveals more.

---

# MS-RPC Enumeration with rpcclient

`rpcclient` can query SMB/RPC services directly and is often effective even with null sessions.

Connect with an empty username (null session attempt):

```bash
rpcclient -U "" 10.129.14.128
```

Useful queries:

| Command                   | Description                   |
| ------------------------- | ----------------------------- |
| `srvinfo`                 | Basic server information      |
| `enumdomains`             | Enumerate domains/workgroups  |
| `querydominfo`            | Domain/server metadata        |
| `netshareenumall`         | Enumerate available shares    |
| `netsharegetinfo <share>` | Details about a share         |
| `enumdomusers`            | Enumerate users               |
| `queryuser <RID>`         | Query user information by RID |

Example: enumerate shares and share permissions metadata:

```text
rpcclient $> netshareenumall
rpcclient $> netsharegetinfo notes
```

If anonymous RPC access is misconfigured, you may see security descriptors (DACLs) that effectively grant broad permissions (for example, `S-1-1-0` which represents **Everyone**). That is a serious risk because it implies visibility or access that should not be public.

---

## User Enumeration and RID Brute Force

If `enumdomusers` is allowed, it can reveal valid usernames for further targeting (password spraying, credential stuffing, targeted phishing in real engagements, etc.). Even if `enumdomusers` is blocked, `queryuser` sometimes works if you know or can guess RIDs.

Example user enumeration:

```text
rpcclient $> enumdomusers
rpcclient $> queryuser 0x3e9
rpcclient $> queryuser 0x3e8
```

RID brute forcing via a loop:

```bash
for i in $(seq 500 1100); do
  rpcclient -N -U "" 10.129.14.128 -c "queryuser 0x$(printf '%x\n' $i)" \
  | grep "User Name\|user_rid\|group_rid" && echo ""
done
```

An Impacket alternative is `samrdump.py`, which automates SAMR-based user discovery:

```bash
samrdump.py 10.129.14.128
```

---

# Additional Enumeration Tools

Different tools can yield different results due to how they implement SMB/RPC calls and parse responses. This is why it is important to use multiple tools and validate findings manually.

## smbmap

```bash
smbmap -H 10.129.14.128
```

## CrackMapExec

```bash
crackmapexec smb 10.129.14.128 --shares -u '' -p ''
```

CrackMapExec is especially useful because it summarises:

* OS and domain hints
* SMB signing state
* SMBv1 availability
* Share permissions (READ/WRITE)

## enum4linux-ng

Install:

```bash
git clone https://github.com/cddmp/enum4linux-ng.git
cd enum4linux-ng
pip3 install -r requirements.txt
```

Enumerate:

```bash
./enum4linux-ng.py 10.129.14.128 -A
```

This tool automates many common checks (NetBIOS names, users, shares, policies, dialects), but you should still validate key results manually, especially if you plan to base exploitation decisions on them.
