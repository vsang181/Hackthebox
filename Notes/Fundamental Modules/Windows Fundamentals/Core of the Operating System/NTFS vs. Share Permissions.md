# NTFS vs Share Permissions (Windows)

Windows dominates corporate environments, which is why it is such a high-value target for attackers. Malware authors follow impact, not ideology. If software can run on an operating system, malicious software can be written for it. Windows is no exception, and the idea that any OS is “immune” is simply incorrect 

As you assess Windows systems, you will repeatedly encounter **network shares**. Many real-world compromises spread laterally through **over-permissive SMB shares**, misconfigured permissions, or legacy protocols such as **SMBv1**, which is still exploited via vulnerabilities like **EternalBlue** on unpatched systems.

Your goal here is to clearly understand **how NTFS permissions and share permissions differ, how they interact, and how they are abused**.

---

## SMB in Simple Terms

Windows uses the **Server Message Block (SMB)** protocol to share resources such as:

* Files
* Folders
* Printers

SMB is everywhere: small offices, large enterprises, and domain environments all rely on it.

Think in client–server terms:

* A **client** requests access to a resource
* A **server** responds and enforces permissions
* Authentication happens locally (workgroup) or centrally (domain / Active Directory)

When you connect to a share, **two permission layers apply at the same time**.

---

## Share Permissions vs NTFS Permissions

These two permission sets are often confused. They are **not the same**, even though they usually apply to the same resource.

### Share Permissions (SMB Layer)

Share permissions control **remote access over the network**.

| Permission   | What it Allows                                   |
| ------------ | ------------------------------------------------ |
| Full Control | Read, write, delete, and change NTFS permissions |
| Change       | Read, write, add, and delete files               |
| Read         | View files and folders only                      |

Share permissions are **coarse-grained**. They are meant to provide simple control at the network boundary.

---

### NTFS Basic Permissions (File System Layer)

NTFS permissions apply **directly on the file system**, regardless of whether access is local or remote.

| Permission           | What it Allows                      |
| -------------------- | ----------------------------------- |
| Full Control         | Full access plus permission changes |
| Modify               | Read, write, add, and delete        |
| Read & Execute       | Read files and run executables      |
| List Folder Contents | View folder contents                |
| Read                 | Read file contents                  |
| Write                | Write data and create files         |
| Special Permissions  | Advanced granular control           |

NTFS permissions are **far more granular** than share permissions.

---

### NTFS Special Permissions (Important for Abuse)

Special permissions are where things get interesting during assessments.

Some high-impact examples:

* **Traverse folder / execute file**
  Access files via full path even if directory listing is denied

* **Create files / write data**
  Create or modify files (useful for dropping payloads)

* **Delete / delete subfolders and files**
  Remove content even without full control

* **Change permissions / take ownership**
  Often leads directly to privilege escalation

These permissions are frequently inherited and misunderstood.

---

## How Permissions Are Evaluated

When accessing a share **over the network**, Windows evaluates:

1. **Share permissions**
2. **NTFS permissions**

The **most restrictive result wins**.

Example:

* Share permission: Read
* NTFS permission: Full Control
* **Effective access: Read**

If either layer denies an action, the action fails.

---

## Local vs Remote Access (Critical Distinction)

* **Local access (console / RDP)**
  → Only **NTFS permissions** apply

* **Remote access (SMB)**
  → **Both NTFS and share permissions** apply

This distinction explains many “why can I access this locally but not over the network?” scenarios.

---

## Enumerating Shares from Linux

From a Linux attack host, you will often use `smbclient`.

### Listing Available Shares

```text
smbclient -L SERVER_IP -U username
```

This shows:

* Default administrative shares (`C$`, `ADMIN$`)
* Custom shares
* IPC endpoints

---

### Connecting to a Share

```text
smbclient '\\SERVER_IP\Company Data' -U username
```

If authentication and permissions are correct, you gain interactive access.

If access fails despite valid credentials, **check the firewall**.

---

## Windows Defender Firewall and SMB

Windows Defender Firewall commonly blocks SMB traffic when:

* The client is not in the same workgroup or domain
* Required inbound rules are disabled
* The active firewall profile is restrictive

Firewall profiles you must remember:

* **Public**
* **Private**
* **Domain**

Best practice is to **enable specific SMB rules**, not disable the firewall entirely. Unfortunately, in real environments, firewalls are often turned off for convenience.

---

## Workgroup vs Domain Authentication

This matters when authenticating to SMB.

* **Workgroup**

  * Authentication uses the local **SAM database**
  * Accounts exist only on that system

* **Domain**

  * Authentication uses **Active Directory**
  * Centralised identity and access control

Always ask yourself: *Where does this account live?*

---

## Mounting a Windows Share from Linux

Once permissions allow access, you can mount the share.

```text
sudo mount -t cifs -o username=user,password=pass //TARGET_IP/"Company Data" /mnt/share
```

If this fails and syntax is correct, install CIFS support:

```text
sudo apt-get install cifs-utils
```

Mounted shares make data exfiltration, file analysis, and persistence much easier.

---

## Built-in Windows Tools for Share Visibility

Windows provides excellent native visibility if you know where to look.

### Viewing Shares

```text
net share
```

You will always see:

* `C$` → Entire system drive
* `ADMIN$` → Windows directory
* `IPC$` → Inter-process communication

These default shares exist on **every Windows system**.

---

### Computer Management

You can also inspect:

* Shares
* Active sessions
* Open files

This is invaluable during incident response or post-exploitation cleanup.

---

### Event Viewer

Windows logs almost everything.

Security logs can reveal:

* Share access
* File operations
* Authentication events

Logs are often ignored by attackers and defenders alike, which makes them extremely valuable.

---

## Why This Matters in Real Attacks

Misconfigured shares enable:

* Lateral movement
* Malware propagation
* Data exfiltration
* Ransomware deployment

Administrators control permissions, which is why they are prime phishing targets. A single bad permission decision can expose an entire organisation.

---

## Key Takeaways for You

* Share permissions and NTFS permissions are **not the same**
* Both apply to SMB access
* The most restrictive permission wins
* NTFS permissions offer far more attack surface
* Default administrative shares exist everywhere
* Firewalls often block SMB before permissions do

If you can confidently reason about **effective permissions**, Windows file sharing stops being confusing and starts becoming predictable—and predictable systems are much easier to break.
