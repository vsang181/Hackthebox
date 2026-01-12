# MSSQL

Microsoft SQL Server (MSSQL) is Microsoft’s **proprietary relational database management system**. Unlike MySQL, MSSQL is closed-source and was originally designed to run on **Windows-based systems**, though modern versions also support Linux and macOS. In real-world environments, MSSQL is still most commonly encountered on **Windows servers**, especially in enterprise networks.

MSSQL integrates tightly with the **.NET ecosystem**, making it a popular choice for applications developed using Microsoft technologies. Because of this deep OS and framework integration, MSSQL often plays a critical role in **Active Directory–backed environments**, which can make it a valuable target during internal and external assessments.

---

## MSSQL Clients

The most common administrative client for MSSQL is **SQL Server Management Studio (SSMS)**. SSMS is a graphical client used for initial configuration, database administration, and long-term maintenance.

Important points regarding SSMS:

* It is a **client-side application**
* It does **not need to be installed on the database server itself**
* It may store **saved credentials** on administrator or developer workstations

This means that a compromised workstation with SSMS installed may provide direct access to production databases.

Other commonly used MSSQL clients include:

* `mssql-cli`
* SQL Server PowerShell
* HeidiSQL
* SQLPro
* Impacket’s `mssqlclient.py`

From a penetration testing perspective, **Impacket’s `mssqlclient.py`** is particularly useful, as Impacket is preinstalled on many offensive security distributions.

To locate it on a typical system:

```bash
locate mssqlclient
```

---

## MSSQL System Databases

MSSQL ships with several **default system databases** that exist on every SQL Server instance. These databases provide insight into the server’s structure and activity.

| Database | Description                                               |
| -------- | --------------------------------------------------------- |
| master   | Stores system-wide configuration and instance metadata    |
| model    | Template used when creating new databases                 |
| msdb     | Used by SQL Server Agent for jobs, alerts, and automation |
| tempdb   | Stores temporary objects and intermediate results         |
| resource | Read-only database containing system objects              |

These databases are often the first place to enumerate once access is obtained.

---

## Default Configuration and Authentication

By default, MSSQL listens on **TCP port 1433**.

When configured for network access:

* The SQL service commonly runs as `NT SERVICE\MSSQLSERVER`
* **Windows Authentication** is often enabled by default
* Encryption is **not enforced unless explicitly configured**

With Windows Authentication, login requests are handled by:

* The **local SAM database**, or
* **Active Directory**, if the host is domain-joined

This centralized authentication is convenient for administrators but introduces risk:
If a domain account with database access is compromised, it can enable **privilege escalation and lateral movement** across the Windows environment.

---

## Dangerous or Commonly Misconfigured Settings

MSSQL is highly configurable, and misconfigurations are common due to operational pressure and complexity. The following areas are frequently abused during attacks:

* MSSQL clients connecting **without enforced encryption**
* Use of **self-signed certificates**, which may be spoofed
* **Named pipes** enabled and exposed over the network
* Weak or default **`sa` account credentials**
* Failure to disable SQL authentication when not required

Even a single oversight in these areas can significantly weaken the security posture of the database server.

---

## Footprinting the MSSQL Service

### Nmap MSSQL Script Scan

Nmap includes several scripts designed specifically for MSSQL. Scanning TCP port 1433 with these scripts can reveal:

* Server hostname
* Instance name
* MSSQL version
* Authentication methods
* Named pipes configuration

Example scan:

```bash
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes \
-sV -p 1433 <IP>
```

This type of scan often provides enough context to decide whether brute forcing, credential reuse, or lateral movement is viable.

---

### MSSQL Enumeration with Metasploit

Metasploit includes an auxiliary module called `mssql_ping`, which can gather similar information quickly.

```text
scanner/mssql/mssql_ping
```

This module typically reveals:

* Server name
* Instance name
* Version
* TCP port
* Named pipe location

Such details should always be documented during reconnaissance.

---

## Connecting with Impacket `mssqlclient.py`

If valid credentials are discovered or guessed, Impacket’s `mssqlclient.py` can be used to establish a remote session and interact with the database using **T-SQL**.

Example using Windows Authentication:

```bash
python3 mssqlclient.py Administrator@<IP> -windows-auth
```

Once connected, it is good practice to enumerate available databases:

```sql
select name from sys.databases;
```

This helps identify:

* Application databases
* Custom or legacy databases
* Potential targets containing sensitive business data
