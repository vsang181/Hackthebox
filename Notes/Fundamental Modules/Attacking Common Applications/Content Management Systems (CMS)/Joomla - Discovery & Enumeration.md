# Joomla Discovery and Enumeration

[Joomla](https://www.joomla.org/) is the third most used CMS on the internet, powering around 3% of all websites and close to 25,000 of the top 1 million sites globally. With over 2.7 million installations tracked through its own statistics API, you will encounter it regularly during assessments. It supports over 7,000 extensions and 1,000 templates, giving it a comparable attack surface to WordPress through third-party components.

***

## Quick Identification

Several files and directories confirm a Joomla installation without any tooling:

- The page source typically contains a `<meta name="generator">` tag identifying Joomla

```bash
curl -s http://dev.inlanefreight.local/ | grep Joomla
```

```
<meta name="generator" content="Joomla! - Open Source Content Management" />
```

- The `robots.txt` file lists Joomla-specific directories including `/administrator/`, `/components/`, `/plugins/`, `/modules/`, and `/templates/`
- The admin login panel is always at `/administrator/` or `/administrator/index.php`
- The `README.txt` file in the root often states the major version:

```bash
curl -s http://dev.inlanefreight.local/README.txt | head -n 5
```

***

## Version Fingerprinting

Joomla exposes its exact version through several files. The most reliable is the manifest XML file in the administrator directory:

```bash
curl -s http://dev.inlanefreight.local/administrator/manifests/files/joomla.xml | xmllint --format -
```

Look for the `<version>` tag in the output:

```xml
<version>3.9.4</version>
<creationDate>March 2019</creationDate>
```

Additional version fingerprinting paths:

| Path | What it reveals |
|---|---|
| `/administrator/manifests/files/joomla.xml` | Exact version number |
| `/plugins/system/cache/cache.xml` | Approximate version |
| `/media/system/js/` | Version from JavaScript files |
| `/README.txt` | Major version branch |

***

## Automated Enumeration with droopescan

[droopescan](https://github.com/droope/droopescan) is a plugin-based CMS scanner with full support for Drupal, WordPress, and SilverStripe, and partial support for Joomla covering version enumeration and interesting URL discovery.

Install via pip:

```bash
sudo pip3 install droopescan
```

Run a Joomla scan:

```bash
droopescan scan joomla --url http://dev.inlanefreight.local/
```

Example output:

```
[+] Possible version(s):
    3.8.10
    3.8.11
    3.8.12
    3.8.13

[+] Possible interesting urls found:
    Detailed version information. - http://dev.inlanefreight.local/administrator/manifests/files/joomla.xml
    Login page.                  - http://dev.inlanefreight.local/administrator/
    License file.                - http://dev.inlanefreight.local/LICENSE.txt
    Version attribute (approx)   - http://dev.inlanefreight.local/plugins/system/cache/cache.xml
```

droopescan returns a version range rather than a pinpoint version. Use the manifest XML file to narrow it down to an exact release.

***

## Automated Enumeration with JoomlaScan

[JoomlaScan](https://github.com/drego85/JoomlaScan) is a Python tool built from the ashes of the OWASP [joomscan](https://github.com/OWASP/joomscan) project. It scans for installed components and extensions against a database of over 600 known Joomla components, locates browsable directories, and finds version-identifying files such as readme, manifest, license, and changelog files. It requires Python 2.7, which you can set up with [pyenv](https://github.com/pyenv/pyenv):

```bash
curl https://pyenv.run | bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
source ~/.bashrc
pyenv install 2.7
pyenv shell 2.7
```

Install the required Python 2.7 dependencies:

```bash
python2.7 -m pip install urllib3
python2.7 -m pip install certifi
python2.7 -m pip install bs4
```

Run the scan:

```bash
python2.7 joomlascan.py -u http://dev.inlanefreight.local
```

JoomlaScan surfaces installed components, flags accessible directories with directory listing enabled, and identifies administrator-only components. It is less useful for version pinpointing but adds depth to component enumeration that droopescan misses.

***

## User Enumeration and Login Behaviour

Unlike WordPress, Joomla returns a generic error message for both invalid usernames and incorrect passwords:

```
Warning: Username and password do not match or you do not have an account yet.
```

This prevents manual username enumeration through login error differences. The default administrator account is `admin`, with the password set at install time.

***

## Brute Forcing the Login

Since user enumeration is not possible through error messages, brute force against the `admin` account is the practical approach. Use the [joomla-bruteforce](https://github.com/ajnik/joomla-bruteforce) script with a wordlist:

```bash
sudo python3 joomla-brute.py -u http://dev.inlanefreight.local -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin
```

```
admin:admin
```

Weak or default credentials are common on forgotten or poorly maintained Joomla installs. Always check `admin:admin`, `admin:password`, and `admin:admin123` before running a full wordlist.

***

## Consolidated Findings

After manual and automated enumeration of the example target, the confirmed data points are:

- Joomla version 3.9.4 confirmed via `/administrator/manifests/files/joomla.xml`
- Admin login portal at `/administrator/index.php`
- User enumeration not possible through login error messages
- Default credentials `admin:admin` confirmed valid
- Multiple installed components identified via JoomlaScan, some with accessible directories
