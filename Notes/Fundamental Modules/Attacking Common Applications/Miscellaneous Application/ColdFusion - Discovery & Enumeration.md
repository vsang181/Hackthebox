## ColdFusion Discovery and Enumeration

[Adobe ColdFusion](https://www.adobe.com/products/coldfusion-family.html) is a Java-based web application platform using the ColdFusion Markup Language (CFML) for rapid development of database-driven applications. Originally developed by Allaire in 1995 and later acquired by Adobe via Macromedia, it remains in active use across government, enterprise, and educational organisations. ColdFusion installations are high-value penetration test targets because they often run as `SYSTEM` on Windows servers and expose an administrator panel that has historically been vulnerable to unauthenticated file reads.

***

## Default Ports

| Port | Protocol | Purpose |
|---|---|---|
| 80 | HTTP | Standard web traffic |
| 443 | HTTPS | Encrypted web traffic |
| 8500 | SSL/HTTP | Primary ColdFusion web server port |
| 1935 | RPC | Remote Procedure Call between components |
| 5500 | Server Monitor | Remote server administration |
| 25 | SMTP | Email functionality |

Port 8500 is the most reliable indicator of ColdFusion during a scan since few other services use it. 

***

## Identification Methods

### Port and Service Scan

Run a full port scan and check for port 8500 with Nmap:

```bash
nmap -p- -sC -Pn 10.129.247.30 --open
```

```
8500/tcp  open  fmtp
```

Browsing to `http://<target>:8500` on a ColdFusion server typically shows a directory listing with two folders: `CFIDE` and `cfdocs`, both of which are created by the ColdFusion installer. 

### Default Admin Path

Navigate to:

```
http://<target>:8500/CFIDE/administrator/index.cfm
```

This is the default ColdFusion Administrator login page.  If it loads, the version number is typically displayed on the login form itself, making version confirmation trivial without authentication.

### Additional Identification Indicators

- File extensions `.cfm` and `.cfc` in page URLs
- HTTP response headers containing `Server: ColdFusion` or `X-Powered-By: ColdFusion`
- Error messages referencing CFML-specific tags such as `cfquery`, `cfinclude`, or `cfdump`
- Default installer files at paths like `/CFIDE/wizards/` and `/cfdocs/`

***

## Notable CVE History

ColdFusion has a substantial vulnerability history, with the most dangerous class being unauthenticated directory traversal flaws that expose the administrator password file directly:

| CVE | Type | Affected Versions |
|---|---|---|
| CVE-2010-2861 | Directory traversal, LFI (unauthenticated) | ColdFusion 8.0, 8.0.1, 9.0, 9.0.1 and earlier  [exploit-db](https://www.exploit-db.com/exploits/14641) |
| CVE-2013-3336 | Directory traversal, arbitrary file read | ColdFusion 10, 9.x and earlier  [acunetix](https://www.acunetix.com/vulnerabilities/web/adobe-coldfusion-directory-traversal/) |
| CVE-2019-15909 | Cross-Site Scripting | ColdFusion 2018, 2016 |
| CVE-2020-24449 | Arbitrary file read | ColdFusion 2016, 2018 |
| CVE-2020-24450 | Command injection | ColdFusion 2016, 2018 |
| CVE-2021-21087 | Arbitrary file upload bypass | ColdFusion 2016, 2018, 2021 |
| CVE-2023-26360 | Arbitrary file read and RCE | ColdFusion 2021 Update 5 and earlier, 2018 Update 15 and earlier  [rapid7](https://www.rapid7.com/blog/post/2023/07/11/cve-2023-29298-adobe-coldfusion-access-control-bypass/) |
| CVE-2023-29298 | Admin access control bypass | ColdFusion 2021, 2018  [rapid7](https://www.rapid7.com/blog/post/2023/07/11/cve-2023-29298-adobe-coldfusion-access-control-bypass/) |

CVE-2023-29298 is particularly significant because it bypasses the IP allowlist protecting `/CFIDE/administrator` by using path variations such as double slashes or trailing dots, and can be chained with CVE-2023-26360 to achieve unauthenticated arbitrary file read even on hardened instances.

***

## CVE-2010-2861: Unauthenticated LFI to Password Disclosure

The most widely exploited historical ColdFusion vulnerability abuses a directory traversal flaw in the administrator login endpoint. The `locale` parameter in `enter.cfm` accepts a relative path that Bash processes before the null byte truncates the `.cfm` extension, allowing file read from anywhere on disk:

```
http://<target>:8500/CFIDE/administrator/enter.cfm?locale=../../../../../../ColdFusion8/lib/password.properties%00en
```

This reads the `password.properties` file, which contains the SHA1-hashed administrator password. Once cracked or passed directly into the login form, full administrator access is gained, which on ColdFusion 8 is a direct path to a webshell via the Scheduled Tasks feature.

The [Exploit-DB entry 14641](https://www.exploit-db.com/exploits/14641) provides a Python script that automates the traversal across multiple possible admin CFM paths in case some are restricted. [exploit-db](https://www.exploit-db.com/exploits/14641)
