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

1. Introduction
  - Introduction

2. WHOIS
  - WHOIS
  - Utilising WHOIS

3. DNS & Subdomains
  - DNS
  - Digging DNS
  - Subdomains
  - Subdomain Bruteforcing
  - DNS Zone Transfers
  - Virtual Hosts
  - Certificate Transparency Logs

4. Fingerprinting
  - Fingerprinting

5. Crawling
  - Crawling
  - robots.txt
  - Well-Known URIs
  - Creepy Crawlies

6. Search Engine Discovery
  - Search Engine Discovery

7. Web Archives
  - Web Archives

8. Automating Recon
  - Automating Recon
