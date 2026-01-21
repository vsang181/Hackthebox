## Automating Recon

While manual reconnaissance can be effective, it does not scale well and is often repetitive. Automating reconnaissance tasks significantly improves speed, consistency, and coverage, enabling security professionals to gather large volumes of intelligence efficiently and with fewer errors.

Automation is especially valuable during the early stages of an engagement, where breadth and visibility matter more than deep exploitation.

---

## Why Automate Reconnaissance?

Automation provides several practical advantages:

* **Efficiency**
  Automated tools execute repetitive tasks far faster than manual workflows, allowing analysts to focus on validation and analysis rather than data collection.

* **Scalability**
  Reconnaissance can be performed across multiple domains, subdomains, or IP ranges simultaneously.

* **Consistency and Repeatability**
  Tools follow predefined logic, ensuring consistent output and making results reproducible across engagements.

* **Comprehensive Coverage**
  Automated frameworks can chain together multiple recon techniques such as DNS enumeration, subdomain discovery, crawling, header analysis, and port scanning.

* **Workflow Integration**
  Many frameworks integrate cleanly with vulnerability scanners, reporting tools, and exploitation frameworks.

---

## Reconnaissance Frameworks

Several frameworks provide an all-in-one approach to automated web reconnaissance:

* **FinalRecon**
  Python-based tool offering modular recon capabilities such as headers, SSL inspection, WHOIS, crawling, DNS, subdomain enumeration, directory brute-forcing, and Wayback Machine analysis.

* **Recon-ng**
  Modular Python framework with database-backed reconnaissance modules covering DNS, hosts, ports, credentials, and third-party data sources.

* **theHarvester**
  Focused on gathering emails, subdomains, employee names, hosts, and banners from public data sources such as search engines and SHODAN.

* **SpiderFoot**
  OSINT automation framework that correlates data across many sources, including DNS, social media, IP intelligence, and breach data.

* **OSINT Framework**
  A curated collection of OSINT tools and resources organised by data type and use case.

---

## FinalRecon Overview

FinalRecon is a lightweight but powerful automated reconnaissance tool that aggregates multiple recon techniques into a single workflow.

### Core Capabilities

* **Header Analysis**

  * Reveals server software, technologies, and misconfigurations.

* **WHOIS Enumeration**

  * Extracts registrar details, domain age, ownership, and contact metadata.

* **SSL/TLS Inspection**

  * Analyses certificate issuer, validity, and configuration details.

* **Crawler**

  * Extracts:

    * HTML, CSS, and JavaScript resources
    * Internal and external links
    * Images, robots.txt, sitemap.xml
    * URLs embedded inside JavaScript
    * Historical URLs from the Wayback Machine

* **DNS Enumeration**

  * Queries over 40 DNS record types, including SPF, DKIM, and DMARC.

* **Subdomain Enumeration**

  * Uses multiple OSINT sources such as:

    * Certificate Transparency logs (crt.sh)
    * Threat intelligence databases
    * Search APIs (VirusTotal, Shodan, CertSpotter, etc.)

* **Directory Enumeration**

  * Brute-forces directories and files using custom wordlists and extensions.

* **Wayback Analysis**

  * Retrieves URLs from archived snapshots spanning several years.

---

## Installing FinalRecon

Clone the repository and install dependencies:

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py
```

Verify installation and view available options:

```bash
./finalrecon.py --help
```

---

## Key Command-Line Options

| Option      | Description                         |
| ----------- | ----------------------------------- |
| `--url`     | Specify the target URL              |
| `--headers` | Retrieve HTTP header information    |
| `--sslinfo` | Extract SSL/TLS certificate details |
| `--whois`   | Perform WHOIS lookup                |
| `--crawl`   | Crawl the target website            |
| `--dns`     | Perform DNS enumeration             |
| `--sub`     | Enumerate subdomains                |
| `--dir`     | Directory and file enumeration      |
| `--wayback` | Retrieve archived URLs              |
| `--ps`      | Fast port scan                      |
| `--full`    | Run full reconnaissance workflow    |

---

## Example Usage

To retrieve HTTP headers and WHOIS information for a target:

```bash
./finalrecon.py --headers --whois --url http://inlanefreight.com
```

This command performs:

* Server and header fingerprinting
* Domain registration analysis
* Automatic result export to disk

Output is saved under the FinalRecon export directory for later analysis and reporting.
