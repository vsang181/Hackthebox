## Subdomain Bruteforcing

Subdomain brute-force enumeration is an **active reconnaissance technique** used to discover valid subdomains by systematically testing potential names against a target domain. This method relies on **predefined wordlists** that contain common or context-specific subdomain names and is highly effective at uncovering hidden or undocumented assets.

Because many organisations follow predictable naming conventions, brute-forcing can reveal environments and services that are not publicly linked but are still reachable.

---

## How Subdomain Bruteforcing Works

The process typically follows these steps:

1. **Wordlist Selection**
   Choosing the right wordlist directly impacts accuracy and noise levels.

   * **General-purpose wordlists**
     Common names such as `dev`, `test`, `admin`, `mail`, `staging`, `api`.
     Useful when no prior knowledge of naming conventions exists.

   * **Targeted wordlists**
     Focused on specific technologies, industries, or organisational patterns (for example, `jira`, `git`, `vpn`, `kibana`).

   * **Custom wordlists**
     Built using keywords gathered from prior reconnaissance such as job postings, documentation, or leaked metadata.

2. **Iteration and Querying**
   Each word from the list is appended to the base domain:

   * `dev.example.com`
   * `staging.example.com`
   * `vpn.example.com`

3. **DNS Resolution**
   A DNS lookup (usually `A` or `AAAA`) is performed to determine whether the subdomain resolves to an IP address.

4. **Filtering and Validation**
   Successfully resolved subdomains are recorded and may be further validated by:

   * HTTP/HTTPS requests
   * Port scanning
   * Technology fingerprinting

---

## Common Tools for Subdomain Bruteforcing

| Tool        | Description                                                                          |
| ----------- | ------------------------------------------------------------------------------------ |
| dnsenum     | Comprehensive DNS enumeration tool supporting brute-force and zone transfer attempts |
| fierce      | Recursive subdomain discovery with wildcard detection                                |
| dnsrecon    | Multi-technique DNS reconnaissance with flexible output                              |
| amass       | Industry-standard subdomain discovery tool with extensive data sources               |
| assetfinder | Lightweight tool for quick subdomain discovery                                       |
| puredns     | High-performance DNS brute-forcing with accurate filtering                           |

---

## DNSEnum

`dnsenum` is a well-established DNS reconnaissance tool written in Perl. It combines multiple techniques into a single workflow, making it useful for structured DNS enumeration.

### Key Capabilities

* **DNS Record Enumeration**
  Retrieves records such as `A`, `AAAA`, `NS`, `MX`, and `TXT`.

* **Zone Transfer Attempts (AXFR)**
  Automatically checks whether name servers allow unauthorised zone transfers.

* **Subdomain Bruteforcing**
  Tests wordlists against the target domain to identify valid subdomains.

* **Search Engine Scraping**
  Extracts additional subdomains from indexed search engine results.

* **Reverse DNS Lookups**
  Identifies other domains hosted on the same IP address.

* **WHOIS Queries**
  Collects domain registration and ownership metadata.

---

## Subdomain Enumeration with dnsenum

The following example demonstrates brute-forcing subdomains for `inlanefreight.com` using a common SecLists wordlist.

```bash
dnsenum --enum inlanefreight.com \
  -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -r
```

### Command Breakdown

* `--enum inlanefreight.com`
  Specifies the target domain and enables multiple enumeration features.

* `-f subdomains-top1million-20000.txt`
  Uses a high-quality wordlist containing the 20,000 most common subdomain names.

* `-r`
  Enables recursive enumeration, allowing discovery of sub-subdomains if a subdomain is found.

---

## Example Output

```text
www.inlanefreight.com.       300 IN A 134.209.24.248
support.inlanefreight.com.   300 IN A 134.209.24.248
```

This confirms that multiple subdomains resolve to active infrastructure, each representing a potential entry point for further reconnaissance and testing.

---

## Practical Considerations

* Brute-force enumeration is **noisy** and may trigger logging or rate-limiting.
* Wildcard DNS configurations can generate false positives and must be filtered.
* Best results are achieved by combining **passive discovery first**, followed by **targeted brute-forcing**.

Subdomain brute-forcing remains one of the most effective techniques for expanding scope during web reconnaissance when used carefully and intelligently.
