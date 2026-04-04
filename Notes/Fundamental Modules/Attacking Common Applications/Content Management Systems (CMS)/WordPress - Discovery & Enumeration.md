# WordPress Discovery and Enumeration

[WordPress](https://wordpress.org/) powers around 32.5% of all websites on the internet, making it the most common CMS you will encounter during external penetration tests. Its plugin and theme ecosystem introduces a large and frequently vulnerable attack surface: roughly 54% of known WordPress vulnerabilities come from plugins, 31.5% from WordPress core, and 14.5% from themes. Getting good at enumerating WordPress installs quickly will pay off consistently throughout your career.

***

## Quick Identification

Several indicators confirm a WordPress installation without any tooling:

- Browsing to `/robots.txt` often reveals `/wp-admin/` and `/wp-content/` directories
- The login page lives at `/wp-login.php`
- Plugins are stored under `/wp-content/plugins/`
- Themes are stored under `/wp-content/themes/`
- The page source usually contains a `<meta name="generator">` tag with the WordPress version

```bash
curl -s http://blog.inlanefreight.local | grep WordPress
```

Expected output:
```
<meta name="generator" content="WordPress 5.8" />
```

***

## Manual Enumeration

### Theme Fingerprinting

Pull the theme name and version directly from stylesheet references in the page source:

```bash
curl -s http://blog.inlanefreight.local/ | grep themes
```

```
href='http://blog.inlanefreight.local/wp-content/themes/business-gravity/assets/vendors/bootstrap/css/bootstrap.min.css'
```

### Plugin Enumeration

Grep for plugin references across multiple pages, since not all plugins load on the homepage:

```bash
curl -s http://blog.inlanefreight.local/ | grep plugins
```

```bash
curl -s http://blog.inlanefreight.local/?p=1 | grep plugins
```

From the page source you can pull version numbers directly out of asset URLs. For example, `?ver=5.4.2` on a Contact Form 7 stylesheet confirms that version is installed. When directory listing is enabled on a plugin folder, browsing to it directly and reading the `readme.txt` file will confirm the version:

```
http://blog.inlanefreight.local/wp-content/plugins/mail-masta/
```

### User Enumeration

WordPress leaks valid usernames through its login error messages. Submitting a valid username with a wrong password returns a different message than submitting a username that does not exist at all. This difference confirms whether a username is registered on the system and makes `/wp-login.php` a reliable enumeration target.

***

## WordPress User Roles

| Role | Access Level |
|---|---|
| Administrator | Full access including adding/deleting users, editing source code |
| Editor | Publish and manage all posts, including other users' posts |
| Author | Publish and manage their own posts only |
| Contributor | Write and manage their own posts but cannot publish |
| Subscriber | Browse posts and edit their own profile |

Administrator access is typically sufficient for code execution on the server. Editor and Author accounts can sometimes access vulnerable plugin functionality that subscriber accounts cannot.

***

## Known Vulnerabilities Found Manually

From manual enumeration of the example target, two critical plugin vulnerabilities stand out:

- [mail-masta 1.0.0](https://www.exploit-db.com/exploits/50226): Unauthenticated Local File Inclusion (LFI), published August 2021
- [wpDiscuz 7.0.4](https://www.exploit-db.com/exploits/49967): Unauthenticated Remote Code Execution via arbitrary file upload ([CVE-2020-24186](https://nvd.nist.gov/vuln/detail/CVE-2020-24186)), published June 2021

Note both findings down and continue enumerating. Do not start exploiting on the first vulnerability found, as there may be easier or more impactful paths still to discover.

***

## Automated Enumeration with WPScan

[WPScan](https://github.com/wpscanteam/wpscan) automates the enumeration of WordPress versions, themes, plugins, and users. It can pull live vulnerability data from the [WPScan Vulnerability Database](https://wpscan.com/) when provided an API token. The free tier allows up to 25 API requests per day, which is enough to cover most standard installations.

Install via RubyGems:

```bash
sudo gem install wpscan
```

Run a full enumeration scan with vulnerability data:

```bash
sudo wpscan --url http://blog.inlanefreight.local --enumerate --api-token dEOFB<SNIP>
```

Key `--enumerate` arguments:

| Argument | Purpose |
|---|---|
| `--enumerate ap` | All plugins |
| `--enumerate at` | All themes |
| `--enumerate u` | Usernames |
| `--enumerate cb` | Config backups |
| Default (no arg) | Vulnerable plugins, themes, users, media, backups |

Use `-t` to control the number of threads (default is 5):

```bash
sudo wpscan --url http://blog.inlanefreight.local --enumerate --api-token <TOKEN> -t 10
```

***

## WPScan Output: Key Findings

Running WPScan against the example target surfaces several important results:

- WordPress 5.8 confirmed as installed and flagged as insecure, with three vulnerabilities: [CVE-2021-39200](https://nvd.nist.gov/vuln/detail/CVE-2021-39200) (data exposure via REST API), [CVE-2021-39201](https://nvd.nist.gov/vuln/detail/CVE-2021-39201) (authenticated XSS in block editor), and a Lodash library issue, all fixed in 5.8.1
- Theme identified as Transport Gravity (a child theme of Business Gravity), not Business Gravity as manual inspection suggested
- mail-masta 1.0 confirmed with two vulnerabilities: unauthenticated LFI and multiple SQL injection
- Users `admin` and `john` both confirmed through author ID brute forcing and login error message analysis
- Directory listing enabled on the upload directory and theme directory
- XML-RPC enabled at `/xmlrpc.php`

XML-RPC being enabled is significant because it allows password brute-forcing via tools like WPScan itself or [Metasploit's wordpress_xmlrpc_login module](https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login), bypassing any login page rate limiting.

***

## Consolidated Findings Summary

After combining manual and automated enumeration, the confirmed data points are:

- WordPress core 5.8 with multiple known vulnerabilities
- Theme: Transport Gravity
- Plugins: [Contact Form 7](https://wordpress.org/plugins/contact-form-7/), mail-masta 1.0 (LFI + SQLi), [wpDiscuz](https://wpdiscuz.com/) 7.0.4 (unauthenticated RCE)
- Valid users: `admin`, `john`
- Directory listing enabled site-wide (sensitive data exposure risk)
- XML-RPC enabled (brute-force vector)

> The combination of manual and automated enumeration is essential. WPScan missed the wpDiscuz and Contact Form 7 plugins in this example. Automated tools supplement manual work but do not replace it.
