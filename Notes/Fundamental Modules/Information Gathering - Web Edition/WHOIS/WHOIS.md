## WHOIS

**WHOIS** is a widely used query and response protocol designed to access databases that store information about registered internet resources. While it is most commonly associated with domain names, WHOIS can also be used to retrieve details about **IP address ranges**, **autonomous systems (ASNs)**, and related ownership metadata. Conceptually, WHOIS functions like a public directory for the internet, allowing us to identify who is responsible for a given online asset.

A basic WHOIS query against a domain might look like the following:

```
whois inlanefreight.com
```

Example output (truncated):

```
Domain Name: inlanefreight.com
Registry Domain ID: 2420436757_DOMAIN_COM-VRSN
Registrar WHOIS Server: whois.registrar.amazon
Registrar URL: https://registrar.amazon.com
Updated Date: 2023-07-03T01:11:15Z
Creation Date: 2019-08-05T22:43:09Z
```

### Common WHOIS Record Fields

A WHOIS record typically contains the following information:

* **Domain Name**
  The registered domain name (for example, `example.com`).

* **Registrar**
  The organisation through which the domain was registered (such as GoDaddy, Namecheap, or Amazon Registrar).

* **Registrant Contact**
  The individual or organisation that owns the domain.

* **Administrative Contact**
  The party responsible for administrative decisions related to the domain.

* **Technical Contact**
  The party responsible for technical configuration and maintenance.

* **Creation and Expiration Dates**
  The date the domain was registered and when it is due to expire.

* **Name Servers**
  DNS servers responsible for resolving the domain to IP addresses.

Depending on the registrar and privacy settings, some of this information may be redacted or replaced with privacy service details.

---

## History of WHOIS

The origins of WHOIS are closely tied to the early development of the internet and the work of **Elizabeth Feinler**, a computer scientist who played a key role in managing network information during the ARPANET era.

In the 1970s, Feinler and her team at the **Stanford Research Institute (SRI) Network Information Center (NIC)** identified the need for a centralised system to track users, hosts, and network resources as ARPANET rapidly expanded. Their solution was the first WHOIS directory: a simple but effective database that catalogued network identifiers and ownership information. This early system laid the groundwork for the modern WHOIS protocol still in use today.

---

## Why WHOIS Matters for Web Reconnaissance

WHOIS data is a valuable source of intelligence during the reconnaissance phase of a penetration test. Even when partially redacted, it can reveal insights that help shape further investigation:

* **Identifying Key Personnel**
  WHOIS records may expose names, email addresses, or organisational details tied to domain ownership. These can be useful for social engineering, phishing campaigns, or understanding organisational structure.

* **Understanding Network Infrastructure**
  Information such as name servers, registrars, and associated IP ranges can indicate hosting providers, DNS configurations, and third-party dependencies, helping map the target’s infrastructure.

* **Historical Analysis**
  By reviewing historical WHOIS records through third-party services, it is possible to track changes in ownership, registrars, or technical contacts. These changes can reveal acquisitions, migrations, or legacy infrastructure that may still be reachable.

As an initial recon technique, WHOIS is low-noise, easy to perform, and often provides context that informs DNS enumeration, subdomain discovery, and broader infrastructure mapping later in the engagement.
