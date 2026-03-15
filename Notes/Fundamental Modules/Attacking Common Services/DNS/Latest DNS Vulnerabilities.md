# Latest DNS Vulnerabilities

Subdomain takeover is one of the most consistently overlooked vulnerabilities in modern web infrastructure. Unlike a software bug that requires a patch, it exists purely because of an administrative oversight -- a DNS record that was never cleaned up after a service was decommissioned. A 2020 study by RedHuntLabs found over 400,000 vulnerable subdomains out of 220 million scanned, with 62% belonging to the e-commerce sector.

## Why It Matters

The primary risk is trust. A subdomain like `customer-drive.inlanefreight.com` inherits the credibility of the parent domain `inlanefreight.com`. A visitor sees a familiar domain name and has no reason to suspect the page is controlled by an attacker. This makes subdomain takeover a particularly effective launchpad for phishing, since the attacker does not need to register a lookalike domain or use typosquatting -- they are operating from the legitimate organisation's own namespace.

Beyond phishing, a taken-over subdomain can be used to:

- Steal session cookies scoped to the parent domain
- Execute cross-site request forgery (CSRF) attacks
- Abuse misconfigured CORS policies that trust the parent domain
- Defeat content security policies (CSP) that whitelist the organisation's own subdomains
- Issue TLS certificates for the subdomain, making attacks appear even more legitimate

Examples of real-world subdomain takeover payouts are documented on the [HackerOne platform](https://hackerone.com/hacktivity?querystring=%22subdomain%20takeover%22), where it is a recognised bug bounty category.

## Mapping to the Concept of Attacks

The attack runs through two cycles.

### Cycle 1 -- Registering the Takeover

| Step | What Happens | Category |
|---|---|---|
| 1 | The attacker discovers a subdomain no longer in use by the company, identified through DNS enumeration | Source |
| 2 | The attacker registers the expired or unclaimed resource on the third-party provider and links it to their own infrastructure | Process |
| 3 | Privileges rest with the primary domain owner, whose DNS records have not been updated -- the third-party provider imposes no restriction on who can claim the resource | Privileges |
| 4 | The attacker's server becomes the destination that the subdomain now resolves to | Destination |

### Cycle 2 -- Triggering the Forwarding

| Step | What Happens | Category |
|---|---|---|
| 5 | A visitor types the subdomain URL into their browser, and the outdated CNAME record in the company's DNS serves as the source | Source |
| 6 | The DNS server finds the CNAME record in its zone data and forwards the visitor to the corresponding domain -- now controlled by the attacker | Process |
| 7 | The DNS administrator's authority to update records is the relevant privilege here -- because the record has not been removed, the DNS server treats the subdomain as legitimate and trusted | Privileges |
| 8 | The visitor is the destination, landing on attacker-controlled infrastructure while believing they are accessing an official company resource | Destination |

## The Root Cause

The underlying issue is an organisational process failure rather than a technical one. When a third-party service is cancelled -- an AWS S3 bucket, a GitHub Pages site, a Heroku app, a Fastly endpoint -- the DNS record pointing to it costs nothing to leave in place and is rarely flagged for cleanup. The result is a CNAME that still exists in the company's authoritative DNS server, pointing to a resource that anyone can now claim.

Prevention requires treating DNS records as assets with a lifecycle. When a service is decommissioned, the corresponding DNS entries should be removed as part of the same process. Regular audits using tools like [Subfinder](https://github.com/projectdiscovery/subfinder) and [can-i-take-over-xyz](https://github.com/EdOverflow/can-i-take-over-xyz) can identify dangling CNAMEs before an attacker does.
