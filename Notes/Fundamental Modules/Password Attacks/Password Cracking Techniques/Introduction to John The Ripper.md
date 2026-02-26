# Introduction to John The Ripper

John the Ripper (JtR) is one of the most widely used offline password cracking tools in penetration testing, originally developed for Unix-based systems in 1996 and now supporting a broad range of platforms including Windows, macOS, BSD, and OpenVMS. The recommended variant for assessment work is the **jumbo** version, which includes performance optimisations, GPU and SIMD acceleration, multilingual wordlists, 64-bit architecture support, and hundreds of additional hash and cipher formats beyond the base release. JtR is pre-installed by default on Kali Linux and most other penetration testing distributions.

## Cracking Modes

JtR offers three primary cracking modes, each suited to different scenarios:

### Single Crack Mode

Single crack mode is a rapid, rule-based technique that derives password candidates from contextual information about the target user rather than an external wordlist. It uses the username, home directory name, and GECOS field values (full name, room number, phone number) from a Linux password file, then applies an extensive set of transformation rules to generate candidate passwords. The underlying assumption is that users frequently base their passwords on personal information such as their own name with common modifications.

Given the following entry in a `passwd` file:

```
r0lf:$6$ues25dIanlctrWxg$nZHVz2z4kCy1760Ee28M1xtHdGoy0C2cYzZ8l2sVa1kIa8K9gAcdBP.GI6ng/qA4oaMrgElZ1Cb9OeXO4Fvy3/:0:0:Rolf Sebastian:/home/r0lf:/bin/bash
```

JtR extracts the username `r0lf`, the real name `Rolf Sebastian`, and the home directory `/home/r0lf`, then generates candidates such as `RolfSebastian`, `Sebastian1`, `r0lfS`, and many others derived from these strings. The attack is run with:

```bash
john --single passwd
```

```
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
1g 0:00:00:00 DONE 1/3 (2025-04-10 07:47) 12.50g/s 5400p/s
[...SNIP...]        (r0lf)
Session completed.
```

Single crack mode is the fastest mode to run and should always be attempted first when a full `/etc/passwd` or `/etc/shadow` file with GECOS data is available.

### Wordlist Mode

Wordlist mode performs a dictionary attack by hashing each entry in a supplied plaintext wordlist and comparing the result against the target hash. It is the most commonly used mode during time-constrained engagements due to its balance of speed and coverage:

```bash
john --wordlist=<wordlist_file> <hash_file>
```

Multiple wordlists can be specified by separating them with a comma. The `--rules` argument applies transformation rules to each wordlist entry before hashing, generating additional candidates through modifications such as capitalising the first letter, appending numbers, substituting characters, and adding special characters at the end. Built-in rule sets like `--rules=best64` or `--rules=KoreLogic` apply collections of commonly effective transformations without requiring a custom rule file.

### Incremental Mode

Incremental mode is JtR's brute-force-style approach. Rather than relying on a predefined wordlist, it generates password candidates dynamically using a statistical model based on Markov chains, which prioritises character combinations that are statistically more likely to appear in real passwords based on training data. This makes it significantly more efficient than naive brute-force while still covering passwords that would never appear in a wordlist:

```bash
john --incremental <hash_file>
```

Character sets and password lengths for incremental mode are configured in `john.conf`. The default modes defined there include:

```bash
[Incremental:ASCII]   # 95 printable ASCII characters, MaxLen = 13
[Incremental:UTF8]    # 196-character UTF-8 set
[Incremental:Latin1]  # 203-character CP1252/ISO-8859-1 set
[Incremental:Custom]  # User-defined charset from custom.chr
```

Incremental mode is the most exhaustive option and is typically reserved as a last resort after single crack and wordlist modes have been exhausted. It can be resource-intensive and may not complete within a practical timeframe for longer or complex passwords.

## Identifying Hash Formats

When a hash's format is unknown, JtR may not identify it with certainty. The tool `hashid` with the `-j` flag provides a list of possible formats alongside the corresponding JtR format string:

```bash
hashid -j 193069ceb0461e1d40d216e32c79c704

[+] MD5 [JtR Format: raw-md5]
[+] MD4 [JtR Format: raw-md4]
[+] LM [JtR Format: lm]
[+] NTLM [JtR Format: nt]
[+] Domain Cached Credentials [JtR Format: mscach]
[+] Domain Cached Credentials 2 [JtR Format: mscach2]
[+] RIPEMD-128 [JtR Format: ripemd-128]
```

When `hashid` returns multiple candidates, the context of where the hash was recovered provides the most reliable indicator of the correct format. A hash extracted from a Windows SAM database is almost certainly NTLM. A hash from `/etc/shadow` prefixed with `$6$` is definitively SHA-512crypt. The `--format` flag is used to specify the format explicitly:

```bash
john --format=raw-md5 --wordlist=rockyou.txt hashes.txt
```

### Supported Hash Formats

JtR supports hundreds of hash formats. The most frequently encountered during assessments are:

| Format | JtR Flag | Common Source |
|---|---|---|
| NT (NTLM) | `--format=nt` | Windows SAM / NTDS.dit |
| MS Cache v2 (DCC2) | `--format=mscash2` | Windows domain-cached credentials |
| SHA-512crypt | `--format=sha512crypt` | Linux `/etc/shadow` (`$6$`) |
| MD5crypt | `--format=md5crypt` | Linux `/etc/shadow` (`$1$`) |
| bcrypt | `--format=bcrypt` | Linux `/etc/shadow` (`$2b$`) |
| NETNTLMv2 | `--format=netntlmv2` | Captured NTLM authentication challenge responses |
| Kerberos 5 | `--format=krb5` | Captured Kerberos AS-REQ tickets |
| MySQL SHA1 | `--format=mysql-sha1` | MySQL user table |
| MS SQL | `--format=mssql` | Microsoft SQL Server |
| RAW MD5 | `--format=raw-md5` | Web application databases |
| RAW SHA256 | `--format=raw-sha256` | Web application databases |

## Cracking Protected Files

JtR includes a suite of conversion tools that extract crackable hashes from password-protected files and convert them into a format JtR can process. The general workflow is:

```bash
<tool> <file_to_crack> > output.hash
john --wordlist=rockyou.txt output.hash
```

Commonly used conversion tools include:

| Tool | Purpose |
|---|---|
| `ssh2john` | Extracts hash from password-protected SSH private keys |
| `zip2john` | Extracts hash from password-protected ZIP archives |
| `rar2john` | Extracts hash from password-protected RAR archives |
| `keepass2john` | Extracts hash from KeePass password database files |
| `pdf2john` | Extracts hash from password-protected PDF documents |
| `office2john` | Extracts hash from password-protected Microsoft Office files |
| `gpg2john` | Extracts hash from password-protected GPG private keys |
| `bitlocker2john` | Extracts hash from BitLocker-encrypted volumes |
| `putty2john` | Extracts hash from password-protected PuTTY private keys |
| `pfx2john` | Extracts hash from PKCS#12 certificate files |
| `hccap2john` | Converts WPA/WPA2 handshake captures |
| `keepass2john` | Extracts hash from KeePass database files |

The full list available on a Kali or Pwnbox installation can be located with:

```bash
locate *2john*
```

This returns both binary tools in `/usr/bin/` and Python or Perl scripts in `/usr/share/john/`, covering formats from 1Password vaults and 7-Zip archives to Android backups and DPAPI master keys.
