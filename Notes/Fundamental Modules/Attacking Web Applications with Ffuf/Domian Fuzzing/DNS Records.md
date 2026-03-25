# DNS Records and Virtual Hosts

When a web application references a hostname like `academy.htb` rather than an IP address, your browser cannot resolve it unless that hostname exists in either a public DNS record or your local `/etc/hosts` file. For lab and internal environments, the hostname will never appear in public DNS, so you need to add it manually.

***

## How DNS Resolution Works

When you type a URL into a browser, it follows this resolution chain:

1. Check the local `/etc/hosts` file for a matching entry
2. Query the configured DNS server (e.g., `8.8.8.8`) for a public record
3. If neither resolves, the connection fails

For HTB labs and internal applications, the domain will never exist in public DNS. Adding it to `/etc/hosts` tells your system to map the hostname directly to the target IP, bypassing DNS entirely.

***

## Adding a Host Entry

```bash
sudo sh -c 'echo "SERVER_IP  academy.htb" >> /etc/hosts'
```

This appends a line to `/etc/hosts` that maps `academy.htb` to the target IP. Once done, any tool or browser on your machine that tries to resolve `academy.htb` will get the target IP immediately without querying DNS.

You can verify the entry was added correctly:

```bash
cat /etc/hosts | grep academy.htb
# SERVER_IP  academy.htb
```

***

## Multiple Hostnames on One IP

A single IP address can serve multiple different websites by using the HTTP `Host` header to determine which site to serve. This is called virtual hosting. When you browse to `academy.htb`, your browser sends:

```http
GET / HTTP/1.1
Host: academy.htb
```

The server reads the `Host` header and decides which application to respond with. This means two hostnames pointing at the same IP can return completely different content, and some of those virtual hosts may never be publicly referenced or linked anywhere.

This is exactly the scenario described: visiting the IP directly and visiting `academy.htb` return the same content, but a message hints that an admin panel exists somewhere under `*.academy.htb`. That admin panel is likely served as a separate virtual host on the same IP, responding only when the correct `Host` header is sent.

***

## Why This Matters for Enumeration

Finding these hidden virtual hosts requires fuzzing the `Host` header with a subdomain wordlist rather than fuzzing URL paths. This is covered in the next section, but the key concept to understand now is:

- IP-based access shows you the default virtual host only
- Other virtual hosts on the same IP are invisible unless you know their hostname
- Developers frequently host admin panels, staging environments, and internal tools as separate subdomains on the same server, assuming obscurity provides security

The `/etc/hosts` workflow also applies to any tool you use, not just browsers. Ffuf, curl, and other tools all respect `/etc/hosts` for resolution, meaning once you add the entry, your entire toolkit can reach the hostname without any additional configuration.

***

## Cleaning Up After an Assessment

When working on multiple targets, `/etc/hosts` can accumulate stale entries. It is good practice to remove entries when you are done with a target:

```bash
# View current entries
cat /etc/hosts

# Remove a specific entry by editing the file
sudo nano /etc/hosts
```

On a shared PwnBox or attack VM, leaving stale entries pointing at recycled IPs can cause confusion on future assessments if a new target reuses the same IP address.
