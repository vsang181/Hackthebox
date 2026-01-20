## Virtual Hosts

Once DNS routes traffic to the correct IP address, the **web server configuration** decides what content is served. Modern web servers (Apache, Nginx, IIS) frequently host **multiple websites or applications on the same server**, and they achieve this through **virtual hosting**.

Virtual hosting allows a single server (often a single IP address) to serve different content based on what the client requested.

---

## How Virtual Hosts Work

The key mechanism behind most virtual hosting is the **HTTP `Host` header**.

When a browser makes a request, it includes a `Host` header indicating the domain it wants:

```http
GET / HTTP/1.1
Host: www.example.com
```

The web server reads the `Host` value and selects the matching **virtual host configuration** (vhost). Each vhost typically has its own:

* `ServerName` / `server_name`
* Document root (content directory)
* Logs
* Proxy rules
* Redirects
* Security controls (auth, IP allowlists, headers)

---

## VHosts vs Subdomains

Although they often overlap in practice, **subdomains** and **virtual hosts** are not the same thing.

### Subdomains

* A DNS concept.
* Usually have DNS records (A/AAAA/CNAME).
* Example: `blog.example.com`

### Virtual Hosts

* A web server concept.
* A configuration that maps a `Host` header to content.
* Can exist **with or without DNS records**.
* Example: `dev.example.com` might exist as a vhost even if DNS does not publish it publicly.

A useful way to remember this:

* **DNS decides where the request goes (IP).**
* **The web server decides what to serve (vhost).**

---

## Accessing a VHost Without DNS

If you know (or suspect) a hostname exists on a target IP, you can still reach it by overriding DNS locally via the **hosts file**.

* Windows: `C:\Windows\System32\drivers\etc\hosts`
* Linux/macOS: `/etc/hosts`

Example entry:

```text
10.10.10.10  dev.example.com
```

Now your browser will send requests to `10.10.10.10`, but with the `Host: dev.example.com` header, which can trigger a different vhost on the same server.

---

## Types of Virtual Hosting

| Type       | How it routes requests          | Pros                             | Cons                                                                                           |
| ---------- | ------------------------------- | -------------------------------- | ---------------------------------------------------------------------------------------------- |
| Name-based | Uses `Host` header              | Scales well, no extra IPs needed | Misconfigurations can expose hidden apps; TLS complexity historically (mostly solved with SNI) |
| IP-based   | Each site has its own IP        | Stronger separation              | Requires multiple IPs; less scalable                                                           |
| Port-based | Each site uses a different port | Useful when IPs are limited      | Less user-friendly; URLs must include ports                                                    |

Name-based virtual hosting is the most common on modern infrastructure.

---

## Why Virtual Hosts Matter for Web Recon

Virtual hosts can significantly expand the attack surface because:

* **Internal or “hidden” apps** may be deployed as vhosts but not publicly advertised in DNS.
* **Staging or development environments** often have weaker security controls.
* Different vhosts can expose:

  * Admin portals
  * Debug endpoints
  * Legacy apps
  * Different authentication rules

This is why **vhost discovery (Host header fuzzing)** is a valuable technique during reconnaissance (with proper authorisation).

---

## Virtual Host Discovery Tools

The general idea is simple:

1. Start with a known target IP (or base domain resolving to an IP).
2. Send repeated HTTP requests while changing the `Host` header.
3. Compare responses to spot vhosts that return different behaviour (status codes, lengths, titles, redirects).

Common tools used for this include:

| Tool        | Strengths                    | Notes                                              |
| ----------- | ---------------------------- | -------------------------------------------------- |
| gobuster    | Fast, widely used            | Reliable for vhost mode with good wordlists        |
| ffuf        | Extremely flexible filtering | Excellent when tuning for response size/word count |
| feroxbuster | Fast and feature-rich        | Good wildcard handling and filtering options       |

---

## Gobuster VHost Fuzzing

### Preparation

* Identify the **target IP / URL**
* Choose a **wordlist** (SecLists is commonly used)
* Know the **base domain** you are trying to append (if relevant)

### Typical command structure

```bash
gobuster vhost -u http://<target_ip_or_host> -w <wordlist> --append-domain
```

Example:

```bash
gobuster vhost -u http://inlanefreight.htb:81 \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  --append-domain
```

Gobuster will report candidates such as:

```text
Found: forum.inlanefreight.htb:81 Status: 200 [Size: 100]
```

### Useful flags

* `-t` to increase threads (speed)
* `-k` to ignore TLS certificate errors (HTTPS targets)
* `-o` to save output

---

## Validating Findings

Vhost fuzzing can produce false positives (especially with wildcard/default vhost behaviour). After a discovery, validate by:

* Visiting the hostname in a browser (via hosts file override if needed)
* Comparing:

  * Status code
  * Response length
  * Title/body content
  * Redirect behaviour
  * Cookies/headers

A “real” vhost usually returns **meaningfully different content** compared to the default site.

---

## Operational Considerations

* Vhost fuzzing can generate noticeable traffic.
* It may trigger WAF/IDS alerting.
* Always tune requests and filters to minimise noise.
* Only perform this activity with explicit authorisation.
