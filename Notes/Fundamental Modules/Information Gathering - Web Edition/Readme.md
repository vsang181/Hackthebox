# Information Gathering - Web Edition

In the vast landscape of the internet, **knowledge is power**. For security professionals, understanding a target’s digital presence is often the difference between guesswork and a well-scoped, evidence-driven assessment. This is where **web reconnaissance** comes in: the systematic process of discovering and analysing publicly accessible (and sometimes unintentionally exposed) information about a website, web application, and the infrastructure supporting it.

Web recon typically focuses on identifying:

* **Attack surface**: domains, subdomains, endpoints, APIs, admin panels, and third-party integrations
* **Technology stack**: web servers, frameworks, CMS platforms, libraries, WAF/CDN usage, authentication flows
* **Content and metadata**: robots.txt, sitemap.xml, JavaScript files, headers, comments, exposed keys/tokens, build artefacts
* **Infrastructure signals**: DNS records, IP ranges, hosting providers, load balancers, TLS certificates, error pages, and banners
* **Security posture indicators**: misconfigurations, exposed directories, weak access controls, debug modes, and outdated components

The goal is not “scan everything blindly,” but to build an accurate map of what exists, how it is connected, and where weak points might realistically appear. Done properly, reconnaissance improves efficiency, reduces noise, and helps ensure testing remains targeted, repeatable, and aligned with scope.

## Table of Contents

1. [Introduction](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Introduction)
  - [Introduction](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Introduction/Introduction.md)

2. [WHOIS](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/WHOIS)
  - [WHOIS](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/WHOIS/WHOIS.md)
  - [Utilising WHOIS](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/WHOIS/Utilising%20WHOIS.md)

3. [DNS & Subdomains](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/DNS%20%26%20Subdomains)
  - [DNS](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/DNS%20%26%20Subdomains/DNS.md)
  - [Digging DNS](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/DNS%20%26%20Subdomains/Digging%20DNS.md)
  - [Subdomains](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/DNS%20%26%20Subdomains/Subdomains.md)
  - [Subdomain Bruteforcing](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/DNS%20%26%20Subdomains/Subdomain%20Bruteforcing.md)
  - [DNS Zone Transfers](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/DNS%20%26%20Subdomains/DNS%20Zone%20Transfers.md)
  - [Virtual Hosts](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/DNS%20%26%20Subdomains/Virtual%20Hosts.md)
  - [Certificate Transparency Logs](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/DNS%20%26%20Subdomains/Certificate%20Transparency%20Logs.md)

4. [Fingerprinting](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Fingerprinting)
  - [Fingerprinting](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Fingerprinting/Fingerprinting.md)

5. [Crawling](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Crawling)
  - [Crawling](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Crawling/Crawling.md)
  - [robots.txt](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Crawling/robots.txt.md)
  - [Well-Known URIs](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Crawling/Well-Known%20URIs.md)
  - [Creepy Crawlies](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Crawling/Creepy%20Crawlies.md)

6. [Search Engine Discovery](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Search%20Engine%20Discovery)
  - [Search Engine Discovery](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Search%20Engine%20Discovery/Search%20Engine%20Discovery.md)

7. [Web Archives](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Web%20Archives)
  - [Web Archives](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Web%20Archives/Web%20Archives.md)

8. [Automating Recon](https://github.com/vsang181/Hackthebox/tree/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Automating%20Recon)
  - [Automating Recon](https://github.com/vsang181/Hackthebox/blob/main/Notes/Fundamental%20Modules/Information%20Gathering%20-%20Web%20Edition/Automating%20Recon/Automating%20Recon.md)
