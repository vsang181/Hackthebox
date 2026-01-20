## Subdomains

When performing DNS reconnaissance, initial focus is often placed on the apex domain (for example, `example.com`). However, real value frequently lies beneath this surface in the form of **subdomains**. Subdomains are extensions of the primary domain and are commonly used to logically separate services, environments, or business functions.

Typical examples include:

* `blog.example.com` – marketing or content platforms
* `shop.example.com` – e-commerce systems
* `mail.example.com` – email infrastructure

From a reconnaissance perspective, each subdomain represents a **potentially distinct attack surface**, often running on different hosts, technologies, or security configurations.

---

## Why Subdomains Matter in Web Reconnaissance

Subdomains frequently expose assets that are not directly linked from the main website and may be overlooked during routine security reviews. These assets often include:

* **Development and Staging Environments**
  Subdomains such as `dev.example.com` or `staging.example.com` are commonly used for testing new features. These environments may have weaker authentication, verbose error messages, debug endpoints, or exposed credentials.

* **Hidden Administrative Interfaces**
  Internal dashboards, CMS panels, or management portals may reside on subdomains that are not intended for public discovery.

* **Legacy Applications**
  Older or deprecated services may still be accessible via subdomains, often running outdated software with known vulnerabilities.

* **Sensitive Data Exposure**
  Misconfigured subdomains may leak internal documents, backups, configuration files, or API endpoints.

Because of this, subdomain discovery is a critical step in mapping the true scope of a target.

---

## Subdomain Enumeration

Subdomain enumeration is the systematic process of identifying all reachable subdomains associated with a target domain.

From a DNS perspective:

* **A / AAAA records** map subdomains to IPv4 or IPv6 addresses.
* **CNAME records** alias subdomains to other domains or cloud services, which may introduce third-party risk or takeover opportunities.

There are two primary approaches to subdomain enumeration: **active** and **passive**.

---

## 1. Active Subdomain Enumeration

Active enumeration involves direct interaction with DNS infrastructure owned by the target.

### Common Active Techniques

* **DNS Zone Transfers (AXFR)**
  A misconfigured authoritative name server may allow zone transfers, revealing all DNS records for a domain. While rare today, this remains a high-impact misconfiguration.

* **Brute-Force Enumeration**
  This approach tests large wordlists of common or context-aware subdomain names against the target domain. Successful resolutions indicate valid subdomains.

  Common tools include:

  * `dnsenum`
  * `ffuf`
  * `gobuster`
  * `dnsrecon`

### Trade-offs

* **Advantages**: High level of control, potential for comprehensive discovery
* **Disadvantages**: Generates traffic, may be logged or rate-limited, higher detection risk

---

## 2. Passive Subdomain Enumeration

Passive enumeration collects subdomain data from third-party sources without querying the target’s infrastructure directly.

### Common Passive Sources

* **Certificate Transparency (CT) Logs**
  SSL/TLS certificates often list subdomains in the Subject Alternative Name (SAN) field. CT logs provide a reliable and historical view of issued certificates.

* **Search Engines**
  Advanced search operators such as `site:example.com` can reveal indexed subdomains.

* **OSINT Aggregators and Databases**
  Public datasets collect DNS and infrastructure information from scans, crawls, and historical records.

### Trade-offs

* **Advantages**: Stealthy, low risk of detection
* **Disadvantages**: Incomplete coverage, may contain stale or inactive subdomains

---

## Enumeration Strategy

No single method provides full visibility. Effective reconnaissance combines both approaches:

* Use **passive enumeration** to build an initial, stealthy baseline.
* Follow up with **active enumeration** to validate, expand, and discover missed assets.

By correlating results from multiple sources, penetration testers can uncover a far more accurate representation of the target’s exposed infrastructure and identify high-value entry points early in the assessment.
