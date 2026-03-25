# Vhost Fuzzing

Vhost fuzzing solves the problem that regular subdomain fuzzing cannot: finding virtual hosts that have no public DNS records. Instead of relying on DNS to resolve a hostname, you send every request directly to a known IP and manipulate the `Host` header to test how the server responds to different hostnames.

***

## The Core Concept

Every HTTP request contains a `Host` header that tells the server which website you are asking for. When a server hosts multiple virtual hosts on one IP, it uses this header to decide which application to respond with. Vhost fuzzing abuses this by cycling through a wordlist as the `Host` value while the destination IP stays constant. DNS is never involved.

```
Normal browser request to academy.htb:
  GET / HTTP/1.1
  Host: academy.htb          <- server serves the main site

Vhost fuzzing with Host: admin.academy.htb:
  GET / HTTP/1.1
  Host: admin.academy.htb    <- server serves the admin vhost (if it exists)
```

***

## The Command

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://academy.htb:PORT/ \
     -H 'Host: FUZZ.academy.htb'
```

The `-H` flag injects a custom header, and `FUZZ` sits inside the header value rather than in the URL path. The URL itself never changes between requests, only the `Host` header does.

***

## The False Positive Problem

The raw output of this scan looks useless at first glance. Every single entry returns `200 OK`, which seems like everything is a valid vhost. This is actually expected behaviour and is not a bug.

What is happening: the server receives an unrecognised `Host` header and falls back to serving its default virtual host, which is `academy.htb` itself. Since that page always exists, every request gets a 200 with the same response body.

The signal you are looking for is a difference in response size. If a valid vhost exists, the server returns that vhost's actual content instead of the default fallback, and the response size will differ from the baseline. In the output above, everything returns `Size: 900`, which is the default page. A valid vhost would show a noticeably different size.

***

## Filtering Out the Noise

Once you identify the baseline response size (900 in this case), you filter it out using the `-fs` flag to exclude responses matching that size:

```bash
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
     -u http://academy.htb:PORT/ \
     -H 'Host: FUZZ.academy.htb' \
     -fs 900
```

After adding `-fs 900`, only responses with a size different from 900 will be shown, cutting through all the false positives and surfacing only genuine vhosts.

***

## Filtering Options in Ffuf

You are not limited to filtering by size. Ffuf gives you several ways to strip out noise:

| Flag | Filters By | Example |
|------|-----------|---------|
| `-fs` | Response size (bytes) | `-fs 900` |
| `-fw` | Word count | `-fw 423` |
| `-fl` | Line count | `-fl 56` |
| `-fc` | Status code | `-fc 302` |
| `-fr` | Regex match in response | `-fr "Not Found"` |

For vhost fuzzing, `-fs` is almost always the right choice because the baseline fallback page will be consistent in size across all false positives.

***

## Vhost vs Subdomain Fuzzing at a Glance

| Factor | Subdomain Fuzzing | Vhost Fuzzing |
|--------|-----------------|--------------|
| Requires DNS | Yes | No |
| Works on internal targets | No | Yes |
| FUZZ position | In the URL | In the `Host` header |
| URL changes per request | Yes | No |
| Finds private vhosts | No | Yes |

The practical takeaway is that vhost fuzzing should always be your go-to method when you already have a target IP, regardless of whether the domain is public or internal. It covers everything subdomain fuzzing does and more.
