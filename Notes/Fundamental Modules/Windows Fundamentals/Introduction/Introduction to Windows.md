# Introduction to Windows

As a penetration tester, you need to be comfortable working across **multiple operating systems**, but in practice, most real-world environments revolve around **Windows and Linux**. Whether the infrastructure is hosted on-premise or in the cloud, these two platforms dominate almost every assessment you will encounter. Understanding how to operate, analyse, attack, and defend both is not optional—it is foundational 

In this section, you focus on **Windows**, its evolution, and the concepts you must understand before moving into deeper enumeration and exploitation.

---

## Why Windows Matters in Assessments

During engagements, you will regularly encounter:

* Windows workstations used by employees
* Windows servers hosting applications and databases
* Active Directory environments managing identities and access
* Windows systems used as jump hosts or attack platforms

Because Windows is designed for **ease of use** and **centralised administration**, it is widely adopted by organisations of all sizes. These same features also introduce complexity—and complexity often leads to misconfiguration.

Your goal here is not to memorise versions or commands, but to **understand how Windows works**, so its behaviour becomes predictable.

---

## A Brief History of Windows

Microsoft first introduced Windows on **20 November 1985** as a graphical shell for **MS-DOS**. Early versions relied heavily on DOS for core functionality, while providing a graphical interface through tools such as:

* Windows File Manager
* Program Manager
* Print Manager

With **Windows 95**, Microsoft tightly integrated Windows and DOS, introduced built-in Internet support, and shipped **Internet Explorer** for the first time. From there, Windows evolved rapidly.

Over the years, Microsoft released many desktop versions, including:

* Windows XP
* Windows Vista
* Windows 7
* Windows 8 / 8.1
* Windows 10
* Windows 11

Each release targeted different audiences, from home users to large enterprises.

---

## Windows Server Evolution

Windows Server followed a parallel but more enterprise-focused path.

* **Windows NT 3.1 Advanced Server (1993)**
  Introduced a more robust architecture and laid the foundation for enterprise Windows.

* **Windows 2000 Server**
  Debuted **Active Directory**, along with the Microsoft Management Console (MMC) and dynamic disk volumes.

* **Windows Server 2003**
  Added server roles, a built-in firewall, and Volume Shadow Copy Service.

* **Windows Server 2008 / 2008 R2**
  Introduced Hyper-V, failover clustering, Server Core, Event Viewer improvements, and major Active Directory enhancements.

* **Windows Server 2012 / 2016 / 2019**
  Added modern virtualisation features, container support, Kubernetes integration, and improved security controls.

As new versions are released, older ones eventually reach **end of life** and stop receiving security updates unless extended support is purchased.

---

## Legacy Windows Systems

In real environments, you will often find **unsupported or legacy Windows versions** still in use. This usually happens because:

* Critical applications depend on old platforms
* Migration costs are high
* Operational risk is prioritised over security

For example, **Windows Server 2008 and 2012** reached end of life in January 2020, yet still appear frequently during assessments. Microsoft has even released emergency patches for deprecated systems in recent years due to critical vulnerabilities such as **EternalBlue (SMBv1)**.

As an assessor, you must understand:

* Which versions are still supported
* What weaknesses exist in older builds
* How behaviour differs between releases

---

## Windows Version Numbers

Windows versions are internally identified by version numbers. You will see these often when enumerating systems.

| Operating System                | Version |
| ------------------------------- | ------- |
| Windows NT 4                    | 4.0     |
| Windows 2000                    | 5.0     |
| Windows XP                      | 5.1     |
| Windows Server 2003 / 2003 R2   | 5.2     |
| Windows Vista / Server 2008     | 6.0     |
| Windows 7 / Server 2008 R2      | 6.1     |
| Windows 8 / Server 2012         | 6.2     |
| Windows 8.1 / Server 2012 R2    | 6.3     |
| Windows 10 / Server 2016 / 2019 | 10.0    |

Knowing these mappings helps you quickly identify the operating system during enumeration.

---

## Identifying Windows Versions with PowerShell

You can retrieve detailed operating system information using **Windows Management Instrumentation (WMI)**.

Example:

```text
Get-WmiObject -Class win32_OperatingSystem | select Version,BuildNumber
```

Sample output:

```text
Version    BuildNumber
-------    -----------
10.0.19041 19041
```

This tells you:

* The Windows version
* The build number
* Whether the system is likely patched or outdated

Other useful WMI classes include:

* `Win32_Process` – running processes
* `Win32_Service` – installed services
* `Win32_Bios` – BIOS and firmware information

You can also query **remote systems**, start and stop services, and gather extensive system metadata using WMI.

---

## Accessing Windows Systems

### Local Access

Local access means interacting with a system directly—keyboard, mouse, and display. This is the most common access method for employee workstations and laptops.

Organisations typically build their security models around the assumption that:

* Users work on company-owned machines
* Devices are physically controlled
* Policies are enforced centrally

---

### Remote Access

Remote access means interacting with a system **over a network**. This is now standard across IT, development, and security roles.

Remote access enables:

* Centralised management
* Standardised tooling
* Automation
* Remote work
* Rapid incident response

Industries such as **MSPs and MSSPs** rely almost entirely on remote access technologies.

Common remote access methods include:

* Virtual Private Networks (VPNs)
* Secure Shell (SSH)
* File Transfer Protocol (FTP)
* Virtual Network Computing (VNC)
* Windows Remote Management (WinRM)
* Remote Desktop Protocol (RDP)

In this module, your focus is primarily on **RDP**.

---

## Remote Desktop Protocol (RDP)

RDP uses a **client–server model**.

* The **client** initiates the connection
* The **server** hosts the desktop session

By default, RDP listens on **TCP port 3389**.

A useful mental model:

* The network is a street
* The IP address is a house
* Ports are doors and windows

If RDP is enabled and reachable, the door is open.

---

## Using RDP from Windows

On Windows systems, you can connect using the built-in **Remote Desktop Connection** client:

```text
mstsc.exe
```

This tool allows you to:

* Connect using IP address or hostname
* Save connection profiles (`.rdp` files)
* Store credentials

As a penetration tester, saved `.rdp` files are valuable artefacts worth searching for during engagements.

---

## Using RDP from Linux

From a Linux attack host, you will frequently use **xfreerdp**.

It is popular because it is:

* Scriptable
* Feature-rich
* Efficient
* Widely supported

You will use `xfreerdp` throughout multiple modules, including file transfer, clipboard sharing, and drive redirection scenarios.

Other RDP clients exist, such as **Remmina** and **rdesktop**, and you are encouraged to experiment and find what works best for you.

---

## What Comes Next

At this point, your goal is familiarity—not mastery.

As you continue, you will:

* Navigate Windows from the command line
* Enumerate users, services, and permissions
* Identify privilege escalation paths
* Move laterally across systems

Treat Windows as a system you are learning to **read**, not fight blindly. Once you understand how it thinks, exploiting it becomes far more systematic and controlled.
