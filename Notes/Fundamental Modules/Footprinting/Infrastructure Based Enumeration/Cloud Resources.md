# Cloud Resources

The adoption of cloud platforms such as AWS, GCP, and Azure has become a core component of modern enterprise infrastructure. Organisations rely on these platforms to enable centralised management, remote access, scalability, and high availability. As a result, cloud services are often deeply integrated into daily operations, development workflows, and internal administration.

Although cloud providers secure the underlying infrastructure, the responsibility for configuration and access control lies with the customer. Misconfigurations introduced by administrators are a frequent source of security issues. Common examples include publicly accessible AWS S3 buckets, Azure Blob Storage containers, or GCP Cloud Storage buckets that lack proper authentication or access restrictions.

---

## Identifying Cloud-Hosted Assets

Cloud storage endpoints are often added to DNS records to simplify access for employees and administrators. This practice can unintentionally expose internal resources during passive reconnaissance.

```bash
for i in $(cat subdomainlist); do
  host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f1,4
done
```

Example output:

```
blog.inlanefreight.com              10.129.24.93
inlanefreight.com                   10.129.27.33
matomo.inlanefreight.com            10.129.127.22
www.inlanefreight.com               10.129.127.33
s3-website-us-west-2.amazonaws.com  10.129.95.250
```

The presence of `s3-website-us-west-2.amazonaws.com` indicates that AWS S3 is being used to host web content or static assets. Such endpoints are often associated with publicly accessible buckets and should be examined carefully during reconnaissance.

---

## Discovering Cloud Storage via Search Engines

In addition to DNS enumeration, search engines can be highly effective for discovering exposed cloud resources. Google Dorks allow searches to be narrowed to specific domains, URLs, or file contents.

Common examples include:

* `inurl:amazonaws.com intext:<company_name>`
* `inurl:blob.core.windows.net intext:<company_name>`

These searches frequently reveal documents such as PDFs, text files, spreadsheets, presentations, or even source code that are hosted directly on cloud storage services.

It is common to encounter cloud-hosted assets that are not linked from the main website but remain publicly accessible. These files often originate from internal documentation, marketing material, or temporary file sharing that was never properly restricted.

---

## Cloud Resources in Website Source Code

Cloud storage endpoints are also commonly referenced directly in website source code. Images, JavaScript files, CSS, and downloadable documents may be loaded from AWS S3, Azure Blob Storage, or GCP Cloud Storage to reduce load on the primary web server.

By inspecting the HTML source of target web pages, it is often possible to identify:

* DNS prefetch or preconnect directives
* External asset URLs pointing to cloud storage
* Hard-coded references to storage containers or buckets

These references can lead directly to additional cloud resources that are not otherwise visible.

---

## Third-Party Intelligence Sources

Third-party reconnaissance platforms can further enrich cloud discovery efforts.

### Domain.Glass

Services such as Domain.Glass provide insight into an organisation’s infrastructure, including DNS records, SSL certificate details, IP address information, and security controls. For example, the presence of Cloudflare with a “Safe” security assessment indicates that a web application firewall or gateway protection layer is in place. This information is useful for mapping defensive controls and understanding how traffic is filtered before reaching backend systems.

---

### GrayHatWarfare

GrayHatWarfare is a powerful platform for passively discovering exposed cloud storage across AWS, Azure, and GCP. It allows searches by:

* Bucket or container name
* Keywords and company abbreviations
* File type (PDF, DOCX, TXT, keys, backups, etc.)

Once a cloud storage resource is identified via Google or DNS, GrayHatWarfare can be used to enumerate its contents and assess what data is publicly accessible.

---

## High-Impact Findings in Cloud Storage

Many organisations use internal abbreviations or shortened company names when naming cloud resources. These naming conventions can be leveraged to discover additional buckets or containers that are not obvious at first glance.

In some cases, sensitive material may be exposed due to human error, particularly under time pressure or poor operational hygiene. Examples of high-risk findings include:

* Configuration files
* Database backups
* API keys and credentials
* SSH private and public keys

The exposure of SSH private keys is especially critical. If a private key is publicly accessible, an attacker may be able to authenticate to internal systems without a password, potentially gaining persistent and wide-ranging access across the environment.

Such findings highlight the importance of strict access controls, continuous monitoring of cloud storage permissions, and regular audits to detect and remediate accidental exposure.
