## Fingerprinting

Fingerprinting is the process of identifying the **technologies, software components, and configurations** that power a website or web application. Just as a physical fingerprint uniquely identifies a person, a web application’s technical signatures can reveal its server software, frameworks, CMS, operating system, and defensive controls.

This information is critical during reconnaissance because it allows both attackers and defenders to understand *what* is running and *how* it might be attacked or protected.

### Why Fingerprinting Matters

Fingerprinting plays a central role in web reconnaissance for several reasons:

* **Targeted Attacks**
  Knowing the exact technologies in use allows attackers to focus on vulnerabilities that specifically affect those versions, frameworks, or platforms.

* **Identifying Misconfigurations**
  Exposed headers, outdated software, default files, or insecure settings often surface during fingerprinting.

* **Prioritising Targets**
  When multiple hosts are in scope, fingerprinting helps identify systems that are outdated, poorly hardened, or more likely to yield results.

* **Building a Complete Attack Profile**
  When combined with DNS, subdomain enumeration, and CT logs, fingerprinting completes the picture of a target’s attack surface.

---

## Fingerprinting Techniques

Common techniques used to fingerprint web technologies include:

* **Banner Grabbing**
  Inspecting banners returned by services (e.g., HTTP, HTTPS) to identify server software and versions.

* **HTTP Header Analysis**
  Headers such as `Server`, `X-Powered-By`, and `Link` often leak information about web servers, frameworks, or CMS platforms.

* **Behavioural Probing**
  Sending crafted requests to trigger error messages or responses unique to certain technologies.

* **Content Analysis**
  Inspecting page source, JavaScript files, paths, comments, or metadata for technology-specific indicators.

---

## Common Fingerprinting Tools

| Tool       | Description                       | Key Features                                          |
| ---------- | --------------------------------- | ----------------------------------------------------- |
| Wappalyzer | Browser extension and web service | Identifies CMSs, frameworks, analytics, and libraries |
| BuiltWith  | Web-based technology profiler     | Detailed stack analysis (free and paid tiers)         |
| WhatWeb    | CLI fingerprinting tool           | Signature-based detection of web technologies         |
| Nmap       | Network and service scanner       | OS and service fingerprinting via NSE scripts         |
| Netcraft   | Online reconnaissance service     | Hosting, technology, and security profiling           |
| wafw00f    | WAF detection tool                | Identifies Web Application Firewalls                  |

---

## Fingerprinting `inlanefreight.com`

To demonstrate fingerprinting in practice, we will analyse the host `inlanefreight.com` using both manual and automated techniques.

---

### Banner Grabbing with `curl`

The first step is to inspect HTTP headers using `curl`:

```bash
curl -I inlanefreight.com
```

Sample output:

```text
HTTP/1.1 301 Moved Permanently
Date: Fri, 31 May 2024 12:07:44 GMT
Server: Apache/2.4.41 (Ubuntu)
Location: https://inlanefreight.com/
Content-Type: text/html; charset=iso-8859-1
```

This reveals:

* Web server: **Apache 2.4.41**
* OS distribution: **Ubuntu**
* HTTP → HTTPS redirection in place

Since the site redirects, we also inspect the HTTPS endpoint:

```bash
curl -I https://inlanefreight.com
```

```text
HTTP/1.1 301 Moved Permanently
Server: Apache/2.4.41 (Ubuntu)
X-Redirect-By: WordPress
Location: https://www.inlanefreight.com/
```

The `X-Redirect-By: WordPress` header strongly suggests the site is running **WordPress**.

Inspecting the final destination:

```bash
curl -I https://www.inlanefreight.com
```

```text
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
Link: <https://www.inlanefreight.com/index.php/wp-json/>; rel="https://api.w.org/"
Content-Type: text/html; charset=UTF-8
```

The presence of `wp-json` confirms a **WordPress REST API**, reinforcing the CMS identification.

---

## WAF Detection with `wafw00f`

Before further probing, it is important to check for a **Web Application Firewall (WAF)**, as it may interfere with scans or block requests.

Install `wafw00f`:

```bash
pip3 install git+https://github.com/EnableSecurity/wafw00f
```

Run detection:

```bash
wafw00f inlanefreight.com
```

Result:

```text
[+] The site https://inlanefreight.com is behind Wordfence (Defiant) WAF.
```

This indicates:

* A **Wordfence WAF** is deployed
* Aggressive probing may be filtered or logged
* Future techniques may require evasion or throttling

---

## Fingerprinting with Nikto

Nikto is primarily a vulnerability scanner, but it also provides valuable fingerprinting data.

Installation (if required):

```bash
sudo apt update && sudo apt install -y perl
git clone https://github.com/sullo/nikto
cd nikto/program
chmod +x nikto.pl
```

Run only fingerprinting-related checks:

```bash
nikto -h inlanefreight.com -Tuning b
```

Key findings from the scan:

* **Server Software**: Apache/2.4.41 (Ubuntu)
* **CMS**: WordPress detected
* **Login Portal**: `/wp-login.php`
* **Missing Security Headers**:

  * `Strict-Transport-Security`
  * `X-Content-Type-Options`
* **Potential Issues**:

  * Outdated Apache version
  * `license.txt` exposed
  * WordPress cookies without `HttpOnly`
  * Possible BREACH attack exposure
