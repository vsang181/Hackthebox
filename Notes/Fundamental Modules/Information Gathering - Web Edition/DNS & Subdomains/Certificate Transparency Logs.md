## Certificate Transparency Logs

In the modern web ecosystem, **trust** is established largely through SSL/TLS. When a browser connects to a website over HTTPS, it relies on a **digital certificate** to verify the site’s identity and encrypt communication. These certificates are issued by trusted third parties known as **Certificate Authorities (CAs)**.

However, certificate issuance is not immune to mistakes or abuse. History has shown that mis-issued or rogue certificates can be used to impersonate legitimate services, perform man-in-the-middle attacks, or silently intercept encrypted traffic. To address this systemic risk, **Certificate Transparency (CT)** was introduced.

---

## What Are Certificate Transparency Logs?

Certificate Transparency (CT) logs are **public, append-only ledgers** that record SSL/TLS certificates issued by CAs. When a certificate is created, it must be submitted to one or more CT logs, where it becomes publicly searchable and verifiable.

You can think of CT logs as a **global audit trail for certificates**. Once a certificate is logged, it cannot be altered or removed without detection.

CT logs serve several important purposes:

* **Early Detection of Rogue Certificates**
  Security teams and domain owners can monitor CT logs to detect certificates issued without their knowledge, allowing rapid revocation before abuse occurs.

* **Accountability for Certificate Authorities**
  Because certificate issuance is public, CAs are held accountable for mistakes or policy violations, strengthening trust in the ecosystem.

* **Improved Web PKI Integrity**
  Public oversight reduces systemic risk in the Web Public Key Infrastructure by making certificate abuse visible rather than opaque.

---

## CT Logs and Web Reconnaissance

From a reconnaissance perspective, CT logs are extremely valuable because they provide **real, historical evidence of subdomains** rather than guesses.

Unlike brute-forcing or wordlist-based enumeration:

* CT logs expose **subdomains that have ever had certificates issued**, even if they are no longer active.
* They often reveal **internal, development, staging, or legacy systems**.
* They are not limited by naming assumptions or wordlist quality.

Certificates frequently include multiple domain names in the **Subject Alternative Name (SAN)** field. This means a single certificate can leak dozens of related subdomains.

Additionally, subdomains found only in expired certificates are often:

* Poorly maintained
* Running outdated software
* Forgotten by administrators

All of which makes them particularly interesting from a security testing perspective.

---

## Searching Certificate Transparency Logs

Several tools and services make CT logs accessible:

| Tool   | Key Features                                      | Use Cases                                     | Pros                      | Cons                  |
| ------ | ------------------------------------------------- | --------------------------------------------- | ------------------------- | --------------------- |
| crt.sh | Simple web interface, JSON output, SAN visibility | Fast subdomain discovery, certificate history | Free, no account required | Limited filtering     |
| Censys | Advanced certificate and host search              | Deep infrastructure analysis                  | Powerful filters, API     | Registration required |

---

## Using `crt.sh` Programmatically

Although `crt.sh` provides a web UI, its JSON output makes it easy to automate searches.

Example: extract subdomains containing `dev` for `facebook.com`:

```bash
curl -s "https://crt.sh/?q=facebook.com&output=json" | \
jq -r '.[] | select(.name_value | contains("dev")) | .name_value' | \
sort -u
```

Sample output:

```text
*.dev.facebook.com
*.newdev.facebook.com
*.secure.dev.facebook.com
dev.facebook.com
devvm1958.ftw3.facebook.com
facebook-amex-dev.facebook.com
facebook-amex-sign-enc-dev.facebook.com
newdev.facebook.com
secure.dev.facebook.com
```

### Breakdown

* `curl -s`: Retrieves CT log data for the target domain in JSON format
* `jq select(.name_value | contains("dev"))`: Filters certificates containing `dev`
* `sort -u`: Deduplicates results

This approach is highly effective for discovering:

* Development environments
* Internal naming conventions
* Infrastructure patterns across large organisations
