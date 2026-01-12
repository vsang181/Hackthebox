## Windows Remote Management Protocols

Modern Windows Server environments are designed to be administered remotely. Since **Windows Server 2016**, remote management is enabled by default, allowing administrators to manage roles, features, and day-to-day operations without logging on locally. Under the hood, Windows remote management relies on services that implement **WS-Management**, hardware management integrations (such as **BMC/IPMI-style** capabilities exposed by vendors), and APIs (COM and scripting interfaces) that allow tools to remotely query and control systems.

During assessments, the most common Windows remote management technologies you will encounter are:

* **Remote Desktop Protocol (RDP)**
* **Windows Remote Management (WinRM)**
* **Windows Management Instrumentation (WMI)**

---

## RDP

**Remote Desktop Protocol (RDP)** is Microsoft’s GUI-based remote access protocol. It transmits display updates and user input between client and server, allowing full interactive control of the remote desktop. RDP typically uses **TCP/3389**, and may also use **UDP/3389** for performance enhancements (depending on configuration).

### Connectivity Requirements

For RDP to be reachable remotely:

* The host firewall and any upstream network firewalls must allow inbound RDP.
* If the target is behind **NAT**, port forwarding must be configured to route traffic to the correct internal host.
* The client must know the server’s reachable IP (public IP if exposed via NAT).

### Encryption and NLA

RDP supports **TLS** (introduced in modern Windows versions) to protect session traffic and the authentication process. In practice, two issues commonly appear:

* Some systems still allow weaker “RDP Security” modes rather than enforcing TLS.
* Certificates are often **self-signed** by default, so clients cannot reliably validate authenticity. This creates opportunity for certificate spoofing and user-warning fatigue in less controlled environments.

On Windows Server, the Remote Desktop service is available by default (but not always enabled). A common baseline hardening measure is **Network Level Authentication (NLA)**, which requires authentication before a full desktop session is created.

### Footprinting RDP

A targeted Nmap scan can reveal:

* Whether **NLA (CredSSP)** is required
* Hostname/domain details via NTLM info
* OS build version indicators

Example:

```
nmap -sV -sC <IP> -p3389 --script rdp*
```

#### Operational Note: RDP Scanning Noise

Using `--packet-trace` demonstrates that Nmap’s RDP probes can include identifiable cookies (for example, `mstshash=nmap`). On hardened networks, this can be detected and used to block scanners or trigger alerts, so be mindful of how aggressively you fingerprint RDP.

### RDP Security Checks

Tools such as `rdp-sec-check.pl` can unauthenticatedly identify the supported RDP security protocols (RDP Security vs TLS vs CredSSP/NLA), based on handshake behaviour. This helps quickly determine whether the server enforces NLA and whether legacy modes are enabled.

### Connecting to RDP

From Linux, common clients include **xfreerdp**, **rdesktop**, and **Remmina**. Example:

```
xfreerdp /u:<user> /p:"<pass>" /v:<IP>
```

Certificate warnings are common in lab and many real environments due to self-signed certificates or hostname mismatches.

---

## WinRM

**Windows Remote Management (WinRM)** is Microsoft’s command-line oriented remote management framework based on **WS-Management** using **SOAP**. It is commonly used for PowerShell remoting, remote administration, and automation workflows.

Default ports:

* **TCP/5985** (HTTP)
* **TCP/5986** (HTTPS)

Many environments expose only **5985** internally, which is functional but provides weaker transport assurances unless additional protections are in place (encryption can still occur at higher layers depending on auth methods, but HTTPS is the cleaner posture).

WinRM enables remote execution via mechanisms such as **WinRS** and PowerShell remoting. It is enabled by default starting with **Windows Server 2012** in many configurations, but older hosts or hardened builds may require explicit enablement and firewall rules.

### Footprinting WinRM

```
nmap -sV -sC <IP> -p5985,5986 --disable-arp-ping -n
```

A typical response on 5985 shows `Microsoft-HTTPAPI/2.0` with a generic “Not Found” page. The presence of the service does not confirm that authentication will succeed—only that the endpoint is reachable.

### Testing and Interaction

* On Windows, `Test-WsMan` can check WS-Man reachability.
* From Linux, **evil-winrm** is commonly used when credentials are available.

Example:

```
evil-winrm -i <IP> -u <user> -p <pass>
```

---

## WMI

**Windows Management Instrumentation (WMI)** is a core Windows administration interface based on Microsoft’s implementation/extension of **CIM/WBEM**. It provides read/write access to a wide range of system settings, telemetry, process and service management, and configuration data. Because of its breadth, WMI is one of the most powerful administrative surfaces in Windows environments.

WMI is typically accessed via:

* PowerShell (WMI/CIM cmdlets)
* VBScript
* WMIC (legacy tooling)
* Remote tooling such as Impacket modules

### Footprinting WMI

WMI commonly begins communication via **TCP/135** (RPC Endpoint Mapper), then negotiates to a **high, dynamic port** for subsequent communication.

Tools like Impacket’s `wmiexec.py` can execute commands remotely when valid credentials are available:

```
wmiexec.py <user>:"<pass>"@<IP> "hostname"
```
