# Drupal Discovery and Enumeration

[Drupal](https://www.drupal.org/) is the third most popular CMS by market share, running on around 1.5% of all websites on the internet, including 56% of government websites worldwide and 7% of the top 10,000 sites. It powers roughly 950,000 tracked instances and is written in PHP with support for MySQL, PostgreSQL, or SQLite as the backend. Like WordPress and Joomla, Drupal supports enhancement through modules and themes, with close to 43,000 modules and 2,900 themes available.

***

## User Roles

Drupal has three default user types:

| Role | Access Level |
|---|---|
| Administrator | Full control over the Drupal site including all settings, users, and content |
| Authenticated User | Can log in and perform operations such as adding and editing articles, within their assigned permissions |
| Anonymous | Default role for all visitors with no account, typically read-only access |

***

## Quick Identification

Several indicators confirm a Drupal installation at a glance:

- The page source contains a `<meta name="Generator">` tag identifying Drupal

```bash
curl -s http://drupal.inlanefreight.local | grep Drupal
```

```
<meta name="Generator" content="Drupal 8 (https://www.drupal.org)" />
<span>Powered by <a href="https://www.drupal.org">Drupal</a></span>
```

- The footer of many Drupal sites displays "Powered by Drupal"
- The `robots.txt` file references `/node`, which is unique to Drupal
- URLs follow the pattern `/node/<nodeid>`, for example `/node/1` for the first post

Drupal uses a [node](https://www.drupal.org/docs/administering-a-drupal-site/managing-content-0/working-with-content-types-and-fields) system to index all content. A node can represent a blog post, article, poll, or any other content type. Browsing to `/node/1` on a suspected Drupal site will often confirm the CMS even when a heavily customised theme hides other visual indicators.

> Not every Drupal installation will expose the login page or allow access to it from the internet.

***

## Version Fingerprinting

The quickest version check is reading the `CHANGELOG.txt` file, which older installations leave publicly accessible:

```bash
curl -s http://drupal-acc.inlanefreight.local/CHANGELOG.txt | grep -m2 ""
```

```
Drupal 7.57, 2018-02-21
```

Newer Drupal installations block access to this file by default and return a 404. When that happens, fall back to automated enumeration.

Additional version fingerprinting paths worth checking manually:

- `/README.txt`
- `/core/CHANGELOG.txt` (Drupal 8+)
- `/core/install.php`
- `/core/modules/system/system.info.yml`

***

## Automated Enumeration with droopescan

[droopescan](https://github.com/droope/droopescan) has significantly more Drupal functionality than it does for Joomla. It enumerates plugins, themes, version ranges, and interesting URLs. Install it via pip if it is not already available:

```bash
sudo pip3 install droopescan
```

Run a scan against the target:

```bash
droopescan scan drupal -u http://drupal.inlanefreight.local
```

Example output:

```
[+] Plugins found:
    php http://drupal.inlanefreight.local/modules/php/
        http://drupal.inlanefreight.local/modules/php/LICENSE.txt

[+] No themes found.

[+] Possible version(s):
    8.9.0
    8.9.1

[+] Possible interesting urls found:
    Default admin - http://drupal.inlanefreight.local/user/login

[+] Scan finished (0:03:19.199526 elapsed)
```

Key things to note from this output:

- The `php` module is present at `/modules/php/`, which is significant since it allows PHP code execution from within Drupal's admin interface
- The version is narrowed to 8.9.0 or 8.9.1
- The admin login is at `/user/login`

Use the version range from droopescan to search [CVE Details](https://www.cvedetails.com/vulnerability-list/vendor_id-1367/product_id-2387/Drupal-Drupal.html) and [Exploit-DB](https://www.exploit-db.com/) for known vulnerabilities affecting those releases. If no direct core exploit applies, focus on installed modules and built-in functionality abuse as the next step.
