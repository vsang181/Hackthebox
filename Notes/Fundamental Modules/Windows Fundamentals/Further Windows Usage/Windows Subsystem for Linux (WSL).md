# Windows Subsystem for Linux (WSL)

The **Windows Subsystem for Linux (WSL)** allows you to run **native Linux binaries directly on Windows**. This means you can use standard Linux tools and workflows without dual-booting or running a traditional virtual machine.

WSL was originally designed for developers who needed access to tools such as `bash`, `sed`, `awk`, `grep`, Ruby, and other Linux command-line utilities while staying inside a Windows environment. From a security and assessment perspective, WSL also introduces an **interesting hybrid environment** that blends Windows and Linux concepts.

---

## WSL Versions and Architecture

There are two major versions of WSL that you should be aware of.

### WSL 1

* Translates Linux system calls into Windows system calls
* Does **not** use a real Linux kernel
* Lightweight and fast for simple command-line tools
* Limited kernel-level functionality

### WSL 2

Released in **May 2019**, WSL 2 introduced a major architectural change.

* Uses a **real Linux kernel**
* Runs inside a lightweight virtual machine
* Built on a subset of **Hyper-V** features
* Provides much better compatibility with Linux applications

From a penetration testing point of view, WSL 2 behaves much more like a real Linux system.

---

## Installing WSL

WSL must be enabled as an **optional Windows feature**. This requires administrative privileges.

You can enable it using PowerShell:

```text
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

Once enabled, you have two main options:

* Install a Linux distribution from the **Microsoft Store**
* Manually download and install a Linux distribution from the command line

Both approaches result in a fully usable Linux environment.

---

## Bash on Windows

When WSL is installed, Windows provides an application called **`bash.exe`**.

You can start a Linux shell simply by typing:

```text
bash
```

This launches a **Bash shell** that behaves like a standard Linux terminal.

Inside this shell, you get:

* A full Linux directory structure
* Standard Linux commands
* Package management
* User accounts and permissions (within WSL)

---

## Linux File System Layout in WSL

Once inside the Bash shell, the file system looks exactly like a Linux system.

Example:

```text
ls /
```

Typical output:

```text
bin  boot  dev  etc  home  lib  lib64  media  mnt
opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

This familiarity makes it easy to switch between Linux systems and WSL without changing habits.

---

## Accessing the Windows File System

One of the most important features of WSL is **seamless file system access**.

Windows drives are mounted under:

```text
/mnt
```

For example:

* `C:\` → `/mnt/c`
* `D:\` → `/mnt/d`

This allows you to:

* Access Windows files from Linux tools
* Use Linux utilities to analyse Windows data
* Move easily between the two environments

This integration is powerful, but it also has security implications if sensitive data is exposed.

---

## Interacting with WSL Like a Linux System

Once inside WSL, you can treat it like any other Linux host.

You can:

* Run standard commands
* Install packages
* Update the system
* Write scripts
* Use Linux enumeration tools

For example, checking kernel information:

```text
uname -a
```

Example output:

```text
Linux WS01 4.4.0-18362-Microsoft #476-Microsoft Fri Nov 01 16:53:00 PST 2019 x86_64 GNU/Linux
```

This confirms that you are running inside a Linux environment backed by Microsoft’s implementation.

---

## Why WSL Matters in Security Work

WSL is not just a convenience feature. From a security perspective, it introduces:

* A Linux execution environment on Windows
* Shared access to the Windows file system
* Additional attack surface if misconfigured
* New persistence and tooling opportunities

During assessments, finding WSL installed on a system should immediately raise questions:

* Who uses it?
* Which distributions are installed?
* What access does it have to sensitive files?
* Is it monitored?

---

## Key Takeaways

* WSL allows native Linux binaries to run on Windows
* WSL 2 uses a real Linux kernel via Hyper-V
* Bash can be launched directly from Windows
* The Linux file system is fully available
* Windows drives are accessible under `/mnt`
* WSL blurs the boundary between Windows and Linux

Understanding WSL helps you operate comfortably in **hybrid environments**, which are increasingly common in real-world Windows infrastructures.
