## Creepy Crawlies

Web crawling can quickly become vast and complex, especially when dealing with modern, dynamic web applications. Fortunately, you do not have to tackle this process manually. A wide range of specialised web crawling tools exist, each designed to automate the discovery of content and extract useful artefacts at scale. These tools significantly reduce manual effort and allow security professionals to focus on analysing results rather than navigating pages by hand.

### Popular Web Crawlers

Several well-established tools are commonly used during web reconnaissance:

* **Burp Suite Spider**
  Burp Suite includes an active crawler known as Spider, designed to map web applications by following links and form actions. It integrates tightly with Burp’s proxy, allowing authenticated crawling, session handling, and seamless transition into manual testing or automated scanning.

* **OWASP ZAP (Zed Attack Proxy)**
  ZAP is a free, open-source web application security scanner that includes both traditional and AJAX-based spiders. It supports automated and manual workflows and is frequently used in CI/CD pipelines due to its scripting and API support.

* **Scrapy (Python Framework)**
  Scrapy is a powerful and extensible Python framework for building custom web crawlers. It excels at extracting structured data, handling complex crawling logic, managing request rates, and exporting results in machine-readable formats. Its flexibility makes it ideal for tailored reconnaissance tasks.

* **Apache Nutch**
  Nutch is a highly scalable crawler written in Java, designed for large-scale crawling across entire domains or even the wider internet. While it requires more configuration and infrastructure, it is well suited for enterprise-level or research-focused crawling projects.

Regardless of the tool used, ethical and responsible crawling is essential. Always obtain explicit authorisation before crawling a target, particularly when performing deep or aggressive scans. Misconfigured crawlers can overload servers, trigger defensive controls, or violate acceptable use policies.

---

## Scrapy for Reconnaissance

In this section, Scrapy is used alongside a custom-built spider designed specifically for reconnaissance. Scrapy’s modular architecture and fine-grained control over requests make it ideal for extracting high-value artefacts such as links, email addresses, JavaScript files, and comments.

For additional background on crawling and traffic inspection, this topic complements proxy-based workflows covered in the *Using Web Proxies* module.

---

## Installing Scrapy

Before building or running any Scrapy-based spiders, ensure Scrapy is installed on your system. Installation can be performed using `pip`:

```
pip3 install scrapy
```

This command installs Scrapy and its dependencies, preparing the environment for custom spider execution.

---

## ReconSpider

ReconSpider is a custom Scrapy-based spider designed for reconnaissance against web applications. It crawls a target domain and extracts commonly useful artefacts for further analysis.

### Download and Setup

Download and extract ReconSpider into your current working directory:

```
wget -O ReconSpider.zip https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip
unzip ReconSpider.zip
```

Once extracted, execute the spider against your target:

```
python3 ReconSpider.py http://inlanefreight.com
```

Replace `inlanefreight.com` with the authorised target domain you wish to crawl.

---

## Output: `results.json`

After execution, ReconSpider saves its findings to a file named `results.json`. This structured output is ideal for post-processing, correlation with other reconnaissance data, or feeding into later assessment phases.

### Example Output Structure

```json
{
    "emails": [
        "lily.floid@inlanefreight.com",
        "cvs@inlanefreight.com"
    ],
    "links": [
        "https://www.themeansar.com",
        "https://www.inlanefreight.com/index.php/offices/"
    ],
    "external_files": [
        "https://www.inlanefreight.com/wp-content/uploads/2020/09/goals.pdf"
    ],
    "js_files": [
        "https://www.inlanefreight.com/wp-includes/js/jquery/jquery-migrate.min.js?ver=3.3.2"
    ],
    "form_fields": [],
    "images": [
        "https://www.inlanefreight.com/wp-content/uploads/2021/03/AboutUs_01-1024x810.png"
    ],
    "videos": [],
    "audio": [],
    "comments": [
        "<!-- #masthead -->"
    ]
}
```

### Extracted Data Explained

| JSON Key         | Description                                                                                  |
| ---------------- | -------------------------------------------------------------------------------------------- |
| `emails`         | Email addresses discovered during crawling, useful for OSINT or social engineering scenarios |
| `links`          | Internal and external URLs identified within the target domain                               |
| `external_files` | Downloadable files such as PDFs or documents, which may contain sensitive metadata           |
| `js_files`       | JavaScript resources that may expose API endpoints, frameworks, or client-side logic         |
| `form_fields`    | HTML form inputs, relevant for identifying authentication and data submission points         |
| `images`         | Image resources, sometimes containing embedded metadata                                      |
| `videos`         | Embedded or hosted video files                                                               |
| `audio`          | Audio resources                                                                              |
| `comments`       | HTML comments that may reveal developer notes or internal structure                          |

---

## Reconnaissance Value

Crawler output is rarely valuable in isolation. Its real strength lies in correlation:

* JavaScript files can be analysed for hidden endpoints or client-side secrets
* Documents may leak usernames, internal paths, or software versions
* Comments can expose legacy functionality or misconfigurations
* Link structures can reveal forgotten or unlinked application areas

Crawling should be treated as a foundational reconnaissance technique that feeds into deeper analysis and exploitation phases, significantly reducing blind spots during a web assessment.
