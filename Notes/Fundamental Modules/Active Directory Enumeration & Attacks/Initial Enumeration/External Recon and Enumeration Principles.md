# External Recon and Enumeration Principles

External reconnaissance is the phase that shapes everything that follows. The information gathered here directly informs which attack paths are worth pursuing internally, what credentials or usernames might already be available, and whether the client's actual external footprint matches what they declared in the scoping document.

## Why Passive Recon Matters Before Active Testing

Passive reconnaissance carries essentially zero risk of detection and zero risk of scope violation, because you are only querying publicly available data sources that have no interaction with the client's systems. This makes it the right place to start every engagement. Beyond the obvious goal of gathering intelligence, it serves two important protective functions: validating that the IP addresses and domains in your scope document actually belong to the client before you touch them, and identifying any infrastructure hosted by third parties (Cloudflare, AWS, Azure) where additional authorization may be required before testing.

The passive phase also frequently surfaces information that makes the internal phase significantly easier. A single set of breached credentials that still work on a VPN portal can eliminate hours of anonymous enumeration. A confirmed email naming convention from LinkedIn cuts the time needed to build a valid username list for password spraying from days to minutes.

## What to Look For

The external recon phase targets five categories of information:

**IP Space and ASN data** tells you who owns what network ranges. The BGP Toolkit at bgp.he.net is the most direct route to this. Enter the organization's domain or a known IP and the tool returns ASN assignments, associated netblocks, DNS records, and hosting providers. For large organizations that own their own ASN, this paints a comprehensive picture of their entire internet footprint. For smaller organizations, it often reveals that their infrastructure runs inside a shared hosting or cloud environment, which has implications for what you are and are not authorized to test.

**Domain and DNS information** built on top of the IP data reveals mail servers, nameservers, and subdomains. Tools like viewdns.info and domaintools provide a fast cross-reference for validating what BGP and DNS queries return. Running nslookup or dig against discovered nameservers adds further verification:

```bash
nslookup ns1.inlanefreight.com
nslookup ns2.inlanefreight.com
```

Subdomains are particularly valuable because they often expose services that were not mentioned in the scoping document but reside on in-scope IP addresses. Bring any such discoveries to the client before testing -- they may want to add them to scope, or they may confirm they are out of bounds.

**Schema and username format** is one of the highest-value items from passive recon because it directly enables password spraying. Contact pages, email signatures in published documents, and LinkedIn profiles often reveal the organization's email naming convention. Finding that Inlanefreight uses `first.last@inlanefreight.com` confirms that AD usernames are likely formatted the same way, giving a wordlist generator like `linkedin2username` the right template to produce a credible username list. Google dorks are efficient for this:

```
intext:"@inlanefreight.com" inurl:inlanefreight.com
filetype:pdf inurl:inlanefreight.com
```

The first searches for email addresses embedded on the organization's own site. The second hunts for PDF documents that may contain metadata, embedded links to internal resources, or references to internal systems and applications.

**Data disclosures** cover any publicly accessible files or code repositories where sensitive information has been inadvertently published. Job postings are a frequently overlooked source -- a SharePoint administrator posting that references specific versions of SharePoint immediately tells you which software and potentially which unpatched vulnerabilities may be present in the environment. GitHub repositories for the organization or its employees can contain hardcoded credentials, API keys, internal hostnames, or connection strings that developers pushed without realizing the sensitivity of the data. Tools like Trufflehog automate the scanning of GitHub history for these exposures, and Grayhat Warfare indexes publicly accessible AWS S3 buckets and Azure Blob storage containers.

**Breach data** from services like Dehashed provides the most immediately actionable intelligence when it produces valid credentials. Searching for the organization's domain returns any email addresses and associated passwords or hashes from known data breaches:

```bash
sudo python3 dehashed.py -q inlanefreight.local -p
```

Even passwords that are old or that do not work on the primary corporate systems are valuable, because people reuse passwords. A credential from a 2018 fitness tracking app breach may still work on a VPN portal, an OWA instance, or an internal Citrix environment that accepts AD authentication. The priority targets to test discovered credentials against are any externally facing services that authenticate against AD: VPN portals, OWA, Office 365, Citrix, RDS gateways, and custom web applications.

## Applying It to Inlanefreight

Walking through the actual enumeration against the module's fictional target illustrates how this process flows in practice:

BGP.he.net returns the IP `134.209.24.248`, mail server `mail1.inlanefreight.com`, and nameservers `NS1` and `NS2`. Viewdns.info confirms the same IP, which is the first validation checkpoint. Running nslookup against the two nameservers returns their IPs (`178.128.39.165` and `206.189.119.186`), adding two new addresses to track and verify against scope before any further action.

The Google dork for PDFs returns one document. Download it immediately and examine it offline. Embedded metadata in Office and PDF files frequently contains the author's full name, their internal username, or references to internal file paths and systems.

The email dork returns the contact page, which lists staff names and emails in `first.last@inlanefreight.com` format. This single finding confirms the email naming convention, which almost certainly mirrors the AD username format (`first.last` or `flast`). Combined with `linkedin2username` scraping the company's LinkedIn for employee names, you can now produce a wordlist that is far more targeted than any generic username list.

## The Iterative Nature of Enumeration

Passive external recon is not a one-time activity that you complete and move on from. It is a resource to return to throughout the engagement whenever internal progress stalls. If anonymous internal enumeration is not yielding results, stepping back to check whether any newly discovered email addresses appear in breach data, or whether a newly found subdomain exposes a login portal, can provide the nudge needed to move forward.

The transition from passive external to active internal happens once you have exhausted public sources and have a clear enough picture of the domain to begin probing it directly. Everything gathered here feeds into the first real internal enumeration steps: building a valid user list, understanding the naming schema, and identifying which externally facing services might accept the credentials found in breach data.
