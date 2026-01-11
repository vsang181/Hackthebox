# Domain Information

Domain information gathering is a fundamental phase of any penetration test. It extends far beyond identifying subdomains and includes understanding the organisation’s complete Internet-facing footprint. The objective is to build a clear picture of how the company operates online, which services it provides, and which technologies, platforms, and architectural components are required to deliver those services reliably and at scale.

This information is typically collected **passively**, without sending direct or intrusive probes to the target infrastructure. By operating as normal users or visitors, we minimise the risk of detection and avoid triggering security controls such as intrusion detection or prevention systems. The OSINT techniques discussed here represent only a small subset of what is possible; more advanced methodologies and strategies are covered in dedicated OSINT and corporate reconnaissance material.

A practical starting point is the organisation’s primary website. By carefully reviewing publicly available content, marketing material, and service descriptions, we can infer the underlying technologies, workflows, and dependencies required to support those offerings. Third-party services can further enrich this view, but understanding the business context should always come first.

For example, many technology-focused companies advertise services such as application development, IoT solutions, hosting, data analytics, or information security consulting. Even if the internal implementation details are not visible, each service implies specific technical requirements, such as backend frameworks, cloud platforms, APIs, CI/CD pipelines, or device management infrastructure. When encountering unfamiliar services, it is essential to research how they typically operate and what technical components they rely on, as this often reveals likely attack surfaces.

This approach aligns closely with the core principles of enumeration: paying attention to both what is visible and what is missing. While we can see the services offered, we cannot directly observe their internal functionality. However, every service is constrained by technical realities. By adopting a developer or system architect’s perspective, we can infer backend components, integrations, and operational patterns that may become relevant during later stages of testing.

---

## Online Presence

Once a baseline understanding of the company and its services has been established, the next step is to assess its overall presence on the Internet. In a typical black-box engagement, we are provided only with a scope and must obtain all additional information independently.

> **Note:** The examples below are illustrative and may differ from lab environments or real-world targets. They are based on real penetration testing scenarios and demonstrate commonly used techniques.

One of the earliest indicators of an organisation’s online footprint is its SSL/TLS certificate. Certificates often contain multiple Subject Alternative Names (SANs), which can reveal additional subdomains that are actively in use.

Example certificate information:

* Validity period: May 18, 2021 – April 6, 2022
* DNS names:

  * inlanefreight.htb
  * [www.inlanefreight.htb](http://www.inlanefreight.htb)
  * support.inlanefreight.htb

Another highly effective source for discovering subdomains is Certificate Transparency (CT) logs. Certificate Transparency is a framework designed to log all publicly trusted certificates issued by certificate authorities, as defined in RFC 6962. This allows domain owners and security researchers to detect misissued or malicious certificates.

Public services such as `crt.sh` aggregate CT log data and provide searchable access to historical and current certificate entries. By querying these logs, we can often identify subdomains that are no longer linked from the main website but may still be active.

---

## Certificate Transparency Enumeration

Certificate Transparency results can also be retrieved in JSON format, which is particularly useful for automation and large-scale analysis.

```bash
curl -s https://crt.sh/?q=inlanefreight.com&output=json | jq .
```

Example entries may include:

* matomo.inlanefreight.com
* smartfactory.inlanefreight.com

To extract a unique list of subdomains from the JSON output, the results can be filtered and deduplicated.

```bash
curl -s https://crt.sh/?q=inlanefreight.com&output=json | jq . \
| grep name | cut -d":" -f2 | grep -v "CN=" \
| cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u
```

This process often reveals a much broader attack surface than initially expected, including legacy systems, monitoring platforms, or development environments.

---

## Identifying Company-Hosted Servers

After identifying candidate subdomains, the next step is to determine which hosts resolve to IP addresses owned or managed directly by the target organisation. This distinction is important, as infrastructure hosted by third-party providers typically requires separate authorisation for testing.

```bash
for i in $(cat subdomainlist); do
  host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f1,4
done
```

Example results:

```
blog.inlanefreight.com        10.129.24.93
inlanefreight.com            10.129.27.33
matomo.inlanefreight.com     10.129.127.22
www.inlanefreight.com        10.129.127.33
```

Hosts that resolve to internal or organisation-owned IP ranges are strong candidates for further investigation.

---

## Shodan Enumeration

Shodan is a search engine that indexes systems permanently connected to the Internet. It identifies open TCP/IP ports and fingerprints exposed services such as HTTP, HTTPS, SSH, FTP, SNMP, Telnet, RTSP, and SIP. This makes it particularly valuable for discovering externally exposed infrastructure, including servers, IoT devices, and industrial systems.

```bash
for i in $(cat subdomainlist); do
  host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f4 >> ip-addresses.txt
done

for i in $(cat ip-addresses.txt); do
  shodan host $i
done
```

Example observations may include:

* Web servers running nginx or Apache
* Exposed SSH services with identifiable versions
* Mail, DNS, or auxiliary management services

Hosts with a larger number of exposed services, such as analytics or monitoring platforms, are often prioritised for active testing due to their expanded attack surface.

---

## DNS Records Enumeration

To further enrich our understanding, we can enumerate all available DNS records for the target domain.

```bash
dig any inlanefreight.com
```

Key DNS record types to analyse include:

* **A records** – Map (sub)domains to IPv4 addresses and confirm Internet-facing hosts
* **MX records** – Identify mail servers responsible for handling email
* **NS records** – Reveal authoritative name servers and often indicate the hosting or registrar provider
* **TXT records** – Commonly used for domain verification and email security mechanisms such as SPF, DKIM, and DMARC

### TXT Record Analysis

Example TXT entries:

```
"MS=ms92346782372"
"atlassian-domain-verification=..."
"google-site-verification=..."
"logmein-verification-code=..."
"v=spf1 include:mailgun.org include:_spf.google.com include:spf.protection.outlook.com ..."
```

Although these records may appear uninteresting at first glance, they often reveal valuable information about third-party services and internal tooling:

* **Atlassian** – Suggests use of platforms such as Jira or Confluence for development and collaboration
* **Google Gmail** – Indicates Google Workspace for email and potentially document storage
* **LogMeIn** – Points to centralised remote access and system administration capabilities
* **Mailgun** – Implies API-driven email services, highlighting potential API attack surfaces
* **Outlook / Office 365** – Suggests Microsoft cloud usage, including OneDrive or Azure resources
* **INWX** – Identified as the domain registrar and DNS provider; verification tokens may correlate with internal account or tenant identifiers

Each of these technologies introduces its own security assumptions, dependencies, and potential attack vectors, which can be explored in more depth during later phases of a penetration test.
