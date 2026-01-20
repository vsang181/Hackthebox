## Introduction

Web reconnaissance is the foundation of any thorough security assessment. It involves the **systematic and methodical collection of information** about a target website or web application before moving into deeper analysis or exploitation. This phase acts as the preparatory groundwork and is a critical component of the **Information Gathering** stage within the broader Penetration Testing Process.

At a high level, the penetration testing lifecycle typically includes:

* Pre-Engagement
* Information Gathering
* Vulnerability Assessment
* Exploitation
* Post-Exploitation
* Lateral Movement
* Proof-of-Concept
* Post-Engagement

Effective reconnaissance directly influences the success of every subsequent phase.

### Objectives of Web Reconnaissance

The primary goals of web reconnaissance include:

* **Identifying Assets**
  Discovering all publicly accessible components associated with the target, such as domains, subdomains, IP addresses, web applications, APIs, and underlying technologies. This provides a complete picture of the target’s online footprint.

* **Discovering Hidden or Exposed Information**
  Identifying unintentionally exposed data such as backup files, configuration files, environment variables, debug endpoints, or internal documentation. These artefacts often reveal sensitive implementation details and potential attack vectors.

* **Analysing the Attack Surface**
  Evaluating the exposed entry points of the target, including web endpoints, authentication mechanisms, third-party integrations, and technology choices. This helps prioritise areas most likely to contain exploitable weaknesses.

* **Gathering Actionable Intelligence**
  Collecting information that can be leveraged for technical exploitation or social engineering, such as employee details, email formats, naming conventions, or technology usage patterns.

From an attacker’s perspective, reconnaissance enables highly targeted and efficient attacks. From a defender’s perspective, the same techniques can be used to proactively identify and remediate weaknesses before they are abused.

---

## Types of Reconnaissance

Web reconnaissance generally falls into two categories: **active** and **passive** reconnaissance. Each approach serves a different purpose and carries different trade-offs in terms of visibility and risk.

---

## Active Reconnaissance

Active reconnaissance involves **direct interaction** with the target system to elicit information. Because traffic is sent to the target, this method carries a higher likelihood of detection.

### Common Active Recon Techniques

| Technique              | Description                                             | Example                                   | Tools                         | Risk of Detection |
| ---------------------- | ------------------------------------------------------- | ----------------------------------------- | ----------------------------- | ----------------- |
| Port Scanning          | Identifying open ports and exposed services             | Scanning ports 80 and 443 on a web server | Nmap, Masscan, Unicornscan    | High              |
| Vulnerability Scanning | Testing for known vulnerabilities and misconfigurations | Scanning for SQL injection or XSS         | Nessus, OpenVAS, Nikto        | High              |
| Network Mapping        | Mapping network paths and infrastructure                | Using traceroute to identify network hops | Traceroute, Nmap              | Medium–High       |
| Banner Grabbing        | Extracting service and version information              | Inspecting HTTP response headers          | Netcat, curl                  | Low               |
| OS Fingerprinting      | Identifying the operating system in use                 | Using Nmap OS detection                   | Nmap, Xprobe2                 | Low               |
| Service Enumeration    | Identifying service versions on open ports              | Detecting Apache or Nginx versions        | Nmap                          | Low               |
| Web Spidering          | Crawling websites to discover content and endpoints     | Enumerating hidden directories            | Burp Suite, OWASP ZAP, Scrapy | Low–Medium        |

Active reconnaissance often provides **high-fidelity and up-to-date information**, but it must be conducted carefully to avoid triggering intrusion detection systems, web application firewalls, or alerting security teams.

---

## Passive Reconnaissance

Passive reconnaissance focuses on gathering information **without directly interacting** with the target systems. It relies entirely on publicly available data sources and is therefore significantly stealthier.

### Common Passive Recon Techniques

| Technique                | Description                                               | Example                                     | Tools                            | Risk of Detection |
| ------------------------ | --------------------------------------------------------- | ------------------------------------------- | -------------------------------- | ----------------- |
| Search Engine Queries    | Leveraging search engines to discover exposed information | Searching for employee data or leaked files | Google, DuckDuckGo, Bing, Shodan | Very Low          |
| WHOIS Lookups            | Retrieving domain registration and ownership data         | Identifying registrants and name servers    | whois, online WHOIS services     | Very Low          |
| DNS Analysis             | Enumerating DNS records and subdomains                    | Discovering mail servers and subdomains     | dig, nslookup, dnsenum, fierce   | Very Low          |
| Web Archive Analysis     | Reviewing historical website snapshots                    | Identifying removed or legacy content       | Wayback Machine                  | Very Low          |
| Social Media Analysis    | Extracting information from public profiles               | Profiling employees and roles               | LinkedIn, Twitter, OSINT tools   | Very Low          |
| Code Repository Analysis | Searching public repositories for leaks                   | Finding exposed credentials in GitHub repos | GitHub, GitLab                   | Very Low          |

Passive reconnaissance is ideal during early engagement phases or when stealth is required, though it may miss information that is only exposed through live interaction.
