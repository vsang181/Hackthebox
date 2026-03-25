# Sub-domain Fuzzing

Sub-domain fuzzing works by placing the `FUZZ` keyword at the subdomain position of a URL and cycling through a wordlist of common subdomain names, checking which ones resolve to a live server. The key distinction from directory fuzzing is that you are testing DNS resolution rather than URL paths.

***

## Public vs Private Targets

The technique works differently depending on whether the target has public DNS records:

- For public domains like `inlanefreight.com`, ffuf sends real HTTP requests and gets responses because each subdomain either resolves via public DNS or does not
- For internal/lab domains like `academy.htb`, the domain has no public DNS record, so every single request fails at the DNS resolution stage before even reaching the server

This is why the scan against `academy.htb` returned 4,997 errors rather than zero results. The errors are all DNS failures, not HTTP 404s. No request ever reached the server.

***

## The Command Structure

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u https://FUZZ.inlanefreight.com/
```

The `FUZZ` keyword replaces the subdomain portion entirely. For a target like `inlanefreight.com`, this generates requests to `support.inlanefreight.com`, `blog.inlanefreight.com`, `www.inlanefreight.com`, and so on down the wordlist.

Wordlist options in SecLists under `/opt/useful/seclists/Discovery/DNS/`:

| Wordlist | Entries | Use Case |
|---------|---------|---------|
| `subdomains-top1million-5000.txt` | 5,000 | Quick initial scan |
| `subdomains-top1million-20000.txt` | 20,000 | Broader coverage |
| `subdomains-top1million-110000.txt` | 110,000 | Thorough assessment |

***

## Why Academy.htb Returns No Results

The `/etc/hosts` entry you added earlier only maps `academy.htb` to the target IP. It says nothing about `admin.academy.htb`, `test.academy.htb`, or any other subdomain. When ffuf tries each of those, the resolution chain is:

1. Check `/etc/hosts` for `admin.academy.htb` - no entry found
2. Query public DNS for `admin.academy.htb` - no record exists
3. Connection fails, ffuf logs an error

This is an important limitation to understand: standard subdomain fuzzing only works against targets with public DNS. For internal targets, you need a different technique entirely, which is virtual host (vhost) fuzzing. Instead of relying on DNS resolution, vhost fuzzing sends requests directly to the known IP and manipulates the HTTP `Host` header to test whether the server responds differently to different hostnames. That approach bypasses DNS completely and is how hidden subdomains on HTB-style targets are discovered.
