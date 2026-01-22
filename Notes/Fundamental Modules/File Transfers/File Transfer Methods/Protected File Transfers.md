## Protected File Transfers

During penetration tests, we often gain access to **highly sensitive data**, such as:

* User lists
* Credentials (for example, `NTDS.dit` for offline password cracking)
* Enumeration data revealing network topology or Active Directory structure

Because of this, it is **critical to protect data during transfer**.

Whenever possible, use **encrypted transport mechanisms** such as:

* SSH / SFTP
* HTTPS

However, these options may not always be available, requiring alternative approaches.

> **Important note**
> Unless explicitly requested by the client, exfiltrating real sensitive data such as:
>
> * Personally Identifiable Information (PII)
> * Financial data (credit card numbers, bank details)
> * Trade secrets
>   is **not recommended**.
>
> When testing DLP or egress controls, use **dummy data** that mimics the real data instead.

Encrypting files **before transfer** helps ensure that even if traffic is intercepted, the data remains unreadable.

Data leakage during an engagement can have serious consequences for:

* The penetration tester
* The consulting company
* The client

Professional and responsible handling of data is mandatory.

---

## File Encryption on Windows

Windows provides multiple ways to encrypt files. One simple and effective method is using the **Invoke-AESEncryption.ps1** PowerShell script.

This script supports:

* AES-256 encryption
* Encrypting strings or files
* Outputting Base64 for encrypted text

---

### Invoke-AESEncryption.ps1 – Usage Examples

**Encrypt a string**

```powershell
Invoke-AESEncryption -Mode Encrypt -Key "p@ssw0rd" -Text "Secret Text"
```

**Decrypt a string**

```powershell
Invoke-AESEncryption -Mode Decrypt -Key "p@ssw0rd" -Text "LtxcRelxrDLrDB9rBD6JrfX/czKjZ2CUJkrg++kAMfs="
```

**Encrypt a file**

```powershell
Invoke-AESEncryption -Mode Encrypt -Key "p@ssw0rd" -Path file.bin
```

**Decrypt a file**

```powershell
Invoke-AESEncryption -Mode Decrypt -Key "p@ssw0rd" -Path file.bin.aes
```

Encrypted files are saved with the `.aes` extension.

---

### Importing the Script

After transferring the script to the target host, import it as a module:

```powershell
Import-Module .\Invoke-AESEncryption.ps1
```

---

### File Encryption Example (Windows)

```powershell
Invoke-AESEncryption -Mode Encrypt -Key "p4ssw0rd" -Path .\scan-results.txt
```

Output:

```
File encrypted to C:\htb\scan-results.txt.aes
```

Directory listing example:

```
Invoke-AESEncryption.ps1
scan-results.txt
scan-results.txt.aes
```

> **Best practice**
> Always use **strong and unique passwords per client**.
> Reusing encryption passwords increases the risk of mass data exposure if one password is compromised.

---

## File Encryption on Linux

Most Linux systems include **OpenSSL**, commonly used for certificate management but also very effective for file encryption.

OpenSSL supports many ciphers. A strong default choice is **AES-256** with PBKDF2 key derivation.

---

### Encrypting a File with OpenSSL

```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc
```

You will be prompted to enter and confirm an encryption password.

---

### Decrypting a File with OpenSSL

```bash
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```

> **Reminder**
> Use a strong, unique password to prevent brute-force attacks if the encrypted file is intercepted.

---

## Secure Transfer Recommendations

Even when files are encrypted, it is still best practice to combine encryption with **secure transport methods**, such as:

* HTTPS
* SFTP
* SSH

Encrypting files provides **defence in depth**, especially when operating in restricted or monitored environments.
