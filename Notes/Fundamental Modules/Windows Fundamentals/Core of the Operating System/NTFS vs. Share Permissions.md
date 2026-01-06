# NTFS vs Share Permissions

Windows dominates desktop operating systems globally, which is exactly why it attracts so much attention from malware authors. From an attacker’s point of view, it is simply the highest-value target. The idea that any operating system is inherently immune to malware is a misconception. If software can be written for an operating system, malicious software can be written for it as well 

A large number of real-world compromises spread through **network shares** that are misconfigured or too permissive. Even today, unpatched systems running **SMBv1** can still be exploited using vulnerabilities such as **EternalBlue**, often leading directly to ransomware incidents.

Your goal here is to clearly understand how **SMB, share permissions, and NTFS permissions** work together, because this interaction is a frequent source of both security failures and privilege escalation opportunities.

---

## SMB in Simple Terms

Windows uses the **Server Message Block (SMB)** protocol to share resources such as:

* Files
* Folders
* Printers

You will encounter SMB in almost every enterprise environment, regardless of size.

You should think of SMB in client–server terms:

* A **client** sends a request
* A **server** processes that request
* The server enforces permissions on the requested resource

Whenever you access a shared folder over the network, **two different permission systems are evaluated**.

---

## Share Permissions vs NTFS Permissions

These two permission sets are often mistaken for being the same. They are not. They frequently apply to the same resource, but they serve different purposes and operate at different layers.

---

## Share Permissions (SMB Layer)

Share permissions control **remote access over the network**.

| Permission   | What it Allows                                   |
| ------------ | ------------------------------------------------ |
| Full Control | Read, write, delete, and change NTFS permissions |
| Change       | Read, add, edit, and delete files and folders    |
| Read         | View file and folder contents only               |

Share permissions are **coarse-grained** and are intended to act as a first line of control at the network boundary.

---

## NTFS Basic Permissions (File System Layer)

NTFS permissions apply **directly to the file system**, whether access is local or remote.

| Permission           | What it Allows                                |
| -------------------- | --------------------------------------------- |
| Full Control         | Complete access, including permission changes |
| Modify               | Read, write, add, and delete                  |
| Read & Execute       | Read file contents and run executables        |
| List Folder Contents | View files and subfolders                     |
| Read                 | Read file contents                            |
| Write                | Create and modify files                       |
| Special Permissions  | Advanced granular controls                    |

NTFS permissions are far more detailed and powerful than share permissions.

---

## NTFS Special Permissions (Where Abuse Happens)

Special permissions are particularly important during assessments.

Some high-impact examples:

* **Traverse folder / execute file**
  Allows access to files via a full path, even when directory listing is denied.

* **Create files / write data**
  Enables dropping payloads or modifying configuration files.

* **Delete subfolders and files**
  Allows removal of content without full control.

* **Change permissions / take ownership**
  Often leads directly to privilege escalation.

These permissions are frequently inherited and misunderstood by administrators.

---

## How Windows Evaluates Permissions

When you access a share **over the network**, Windows evaluates permissions in this order:

1. **Share permissions**
2. **NTFS permissions**

The **most restrictive result always wins**.

Example:

* Share permission: Read
* NTFS permission: Full Control
* **Effective access: Read**

If either layer blocks an action, that action fails.

---

## Local Access vs Network Access (Critical Difference)

You must always distinguish between these two scenarios:

* **Local access (console or RDP)**
  → Only **NTFS permissions** apply

* **Network access (SMB)**
  → **Both share and NTFS permissions** apply

This explains why a user may be able to access a folder locally but not over the network.

---

## Enumerating SMB Shares from Linux

From a Linux attack host, you will typically use `smbclient`.

### Listing Available Shares

```text
smbclient -L SERVER_IP -U username
```

This lists:

* Default administrative shares (`C$`, `ADMIN$`)
* Custom user-created shares
* IPC endpoints

---

### Connecting to a Share

```text
smbclient '\\SERVER_IP\Company Data' -U username
```

If authentication and permissions are correct, you receive an interactive SMB shell.

If this fails despite valid credentials, your next check should always be the firewall.

---

## Windows Defender Firewall and SMB

Windows Defender Firewall commonly blocks SMB connections when:

* The client is not part of the same workgroup or domain
* Required inbound rules are disabled
* The active firewall profile is restrictive

Firewall profiles you must remember:

* **Public**
* **Private**
* **Domain**

Best practice is to enable **specific SMB rules**, not disable the firewall entirely. Unfortunately, in real environments, firewalls are often turned off for convenience.

---

## Workgroup vs Domain Authentication

Authentication behaviour depends on where the account exists.

* **Workgroup**

  * Authentication uses the local **SAM database**
  * Accounts exist only on that single system

* **Domain**

  * Authentication uses **Active Directory**
  * Centralised identity and access control

Before attempting authentication, always ask yourself: *Where does this account live?*

---

## Mounting a Windows Share from Linux

Once permissions allow access, you can mount the share.

```text
sudo mount -t cifs -o username=user,password=pass //TARGET_IP/"Company Data" /mnt/share
```

If this fails and your syntax is correct, install CIFS support:

```text
sudo apt-get install cifs-utils
```

Mounted shares make analysis, persistence, and data exfiltration much easier.

---

## Built-in Windows Visibility Tools

Windows provides strong native tooling to monitor shared resources.

### Viewing Shares

```text
net share
```

You will always see default administrative shares such as:

* `C$` – entire system drive
* `ADMIN$` – Windows directory
* `IPC$` – inter-process communication

These exist on **every Windows system** by default.

---

### Computer Management

This interface allows you to inspect:

* Shared folders
* Active SMB sessions
* Open files

This is extremely useful during incident response or post-exploitation analysis.

---

### Event Viewer

Windows logs nearly everything.

Security logs can reveal:

* Share access events
* File operations
* Authentication attempts

Logs are often ignored, which makes them one of the most valuable forensic resources.

---

## Why This Matters in Real Attacks

Misconfigured shares are commonly abused for:

* Lateral movement
* Malware propagation
* Data exfiltration
* Ransomware deployment

Administrators control permissions, which is why they are frequently targeted in spear-phishing campaigns. A single poor permission decision can expose an entire organisation.

---

## Key Points to Remember

* Share permissions and NTFS permissions are **not the same**
* Both apply to SMB access
* The most restrictive permission wins
* NTFS permissions provide far more attack surface
* Default administrative shares exist everywhere
* Firewalls often block SMB before permissions do

Once you can reason confidently about **effective permissions**, Windows file sharing becomes predictable—and predictable systems are much easier to break and defend.
