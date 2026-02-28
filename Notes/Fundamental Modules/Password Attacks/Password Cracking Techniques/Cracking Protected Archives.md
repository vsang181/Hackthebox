# Cracking Protected Archives

Password-protected archives are one of the most commonly encountered protected file types during post-exploitation. Employees routinely bundle sensitive documents into ZIP, RAR, or 7-Zip archives before transferring them, and in enterprise environments, virtual drives encrypted with BitLocker are frequently used to protect entire folders of confidential data on shared or company-issued devices.

## Archive Format Overview

Not all archive formats support native password protection. ZIP, RAR, and 7-Zip have built-in encryption capabilities, while formats such as TAR and GZIP do not and are instead commonly encrypted externally using `openssl` or `gpg`. The `file` command is the most reliable way to determine the actual format of an archive regardless of its extension:

```bash
file GZIP.gzip
GZIP.gzip: openssl enc'd data with salted password
```

A comprehensive list of archive file extensions can be queried dynamically from [FileInfo](https://fileinfo.com/filetypes/compressed):

```bash
curl -s https://fileinfo.com/filetypes/compressed | html2text | awk '{print tolower($1)}' | grep "\." | tee -a compressed_ext.txt
```

At the time of writing, the site lists over 365 compressed file types.

## Cracking ZIP Archives

The ZIP format is the most commonly encountered password-protected archive in Windows environments. The workflow follows the standard hash-extraction-then-crack pattern:

```bash
zip2john ZIP.zip > zip.hash
cat zip.hash

ZIP.zip/customers.csv:$pkzip2$1*2*2*0*2a*1e*490e7510*0*42*...$:customers.csv:ZIP.zip::ZIP.zip
```

Once the hash is extracted, JtR processes it with the desired wordlist:

```bash
john --wordlist=rockyou.txt zip.hash
john zip.hash --show

ZIP.zip/customers.csv:1234:customers.csv:ZIP.zip::ZIP.zip
1 password hash cracked, 0 left
```

Hashcat can also be used with mode `17200` (PKZIP) or `13600` (WinZip AES) depending on the encryption method used. The `file` output and the hash prefix produced by `zip2john` will indicate which applies.

## Cracking OpenSSL-Encrypted GZIP Files

When `openssl` is used to encrypt a GZIP file, the resulting file contains no crackable hash structure that tools like JtR can work with directly. Instead, the correct approach is to attempt live decryption in a loop, relying on the success or failure of the decompression step as the oracle:

```bash
for i in $(cat rockyou.txt); do
    openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null | tar xz
done
```

The loop will produce repeated `gzip: stdin: not in gzip format` errors for every incorrect password, which can be ignored. When the correct password is tried, the archive extracts silently and the output file appears in the current directory:

```bash
ls
customers.csv  GZIP.gzip  rockyou.txt
```

This approach is significantly slower than GPU-accelerated hash cracking because each attempt involves a full decryption and decompression cycle on the CPU. Restricting the wordlist to a targeted, context-specific list is therefore important for keeping this technique practical within an engagement timeframe.

## Cracking BitLocker-Encrypted Drives

[BitLocker](https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-device-encryption-overview-windows-10) is Microsoft's full-disk encryption feature, available since Windows Vista, using AES with either 128-bit or 256-bit keys. In enterprise environments, BitLocker-encrypted virtual hard drives (`.vhd` / `.vhdx`) are commonly used to store sensitive documents, credentials, or configuration notes on company devices. When encountered during post-exploitation, `bitlocker2john` extracts four distinct hashes from the volume:

- `$bitlocker$0$...` — Password hash (attack target)
- `$bitlocker$1$...` — Password hash (alternative format)
- `$bitlocker$2$...` — Recovery key hash
- `$bitlocker$3$...` — Recovery key hash (alternative format)

The 48-digit recovery key is randomly generated and impractical to brute-force unless partial knowledge is available. The password hash is the realistic target:

```bash
bitlocker2john -i Backup.vhd > backup.hashes
grep "bitlocker\$0" backup.hashes > backup.hash
```

Hashcat mode `22100` handles the `$bitlocker$0$` hash. BitLocker's AES encryption combined with its PBKDF2-based key derivation makes it extremely slow to crack — on a consumer CPU this may be as low as 25 hashes per second, as visible in the example output below:

```bash
hashcat -a 0 -m 22100 '$bitlocker$0$16$02b329c0...' /usr/share/wordlists/rockyou.txt
```

```
Speed.#1.........:       25 H/s (9.28ms)
Status...........: Cracked
Hash.Mode........: 22100 (BitLocker)
Time.Started.....: Sat Apr 19 17:49:25 2025 (1 min, 56 secs)
$bitlocker$0$...:1234qwer
```

A targeted wordlist is essential here given the cracking speed. A 14-million-entry list like rockyou.txt would take well over a week at 25 H/s on CPU hardware.

## Mounting the Decrypted Drive

### Windows

On Windows, double-clicking the `.vhd` file mounts it as a virtual disk. BitLocker will display an access error until the volume is unlocked. Double-clicking the mounted drive prompts for the password.

### Linux

[Dislocker](https://github.com/Aorimn/dislocker) handles BitLocker decryption on Linux and macOS. The workflow requires two mount points — one for the decrypted container produced by Dislocker, and one for the actual filesystem within it:

```bash
# Install dislocker
sudo apt-get install dislocker

# Create mount points
sudo mkdir -p /media/bitlocker
sudo mkdir -p /media/bitlockermount

# Attach VHD as a loop device, decrypt, and mount
sudo losetup -f -P Backup.vhd
sudo dislocker /dev/loop0p2 -u1234qwer -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount

# Browse the decrypted contents
cd /media/bitlockermount && ls -la
```

The `-u` flag passes the password directly. If the filesystem type is not detected automatically during mounting, specifying `-t ntfs` or `-t ntfs-3g` explicitly in the `mount` command resolves the issue on most systems.

When analysis is complete, unmount in reverse order:

```bash
sudo umount /media/bitlockermount
sudo umount /media/bitlocker
```

## Archive Type to Tool Mapping

| Archive Type | Hash Extraction | Hashcat Mode | Notes |
|---|---|---|---|
| ZIP (PKZIP) | `zip2john` | `17200` | Most common; fast to crack |
| ZIP (WinZip AES) | `zip2john` | `13600` | AES-based; slower |
| RAR3 | `rar2john` | `12500` | MD5-based derivation |
| RAR5 | `rar2john` | `13000` | SHA-256-based; slower |
| 7-Zip | `7z2john.pl` | `11600` | AES-256 + SHA-256; very slow |
| BitLocker | `bitlocker2john` | `22100` | Password hash; ~25 H/s on CPU |
| OpenSSL GZIP | Direct loop | N/A | No hash extraction; live decryption oracle |
| GPG | `gpg2john` | `17010` | Depends on cipher and key type |
