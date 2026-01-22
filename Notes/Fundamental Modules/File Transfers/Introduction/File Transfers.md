## File Transfers

File transfers are a common requirement during security assessments. Whether transferring tools to a target system or exfiltrating data for analysis, understanding **multiple file transfer techniques** is essential, especially in restricted environments.

---

## Setting the Stage

Consider the following engagement scenario:

* Remote Code Execution (RCE) is achieved on an **IIS web server** via an unrestricted file upload vulnerability.
* An initial **web shell** is uploaded, followed by a **reverse shell** to enumerate the system.
* An attempt is made to transfer `PowerUp.ps1` using **PowerShell**, but this fails due to an **Application Control Policy** blocking PowerShell.
* Manual enumeration reveals the presence of **SeImpersonatePrivilege**, indicating a viable privilege escalation path.
* A binary (for example, **PrintSpoofer**) must be transferred to escalate privileges.
* Attempts to download the binary using **Certutil** fail due to **web content filtering** blocking access to services such as GitHub, Dropbox, and Google Drive.
* An **FTP server** is set up, but outbound **TCP port 21** is blocked by the firewall.
* An **SMB server** is created using the Impacket `smbserver` tool.
* Outbound **TCP port 445** is allowed, enabling the binary to be transferred successfully.
* Privilege escalation is completed, resulting in **administrator-level access**.

This scenario highlights the importance of flexibility and adaptability when transferring files in constrained environments.

---

## Why File Transfer Techniques Matter

Understanding file transfer mechanisms and network behavior helps overcome defensive controls such as:

* Application whitelisting
* Antivirus (AV) and Endpoint Detection and Response (EDR)
* Firewalls restricting outbound ports
* Intrusion Detection and Prevention Systems (IDS/IPS)

File transfer operations are often monitored or restricted, making it critical to have **multiple fallback options** available.

---

## Operating System Support

File transfer is a **core capability** of both Windows and Linux operating systems. While many tools exist to support this, administrators may block or heavily monitor commonly abused utilities. As a result, relying on a single method is rarely sufficient.

This module focuses on techniques that:

* Use **built-in or commonly available tools**
* Are practical in real-world enterprise environments
* Can bypass common host-based and network-based restrictions

---

## Scope of This Module

This module covers:

* File transfer techniques for **Windows and Linux**
* Methods suitable for restricted or monitored networks
* Practical approaches commonly required in other HTB Academy modules

The techniques presented are **not exhaustive**, but they provide a strong foundation that can be reused across assessments.
