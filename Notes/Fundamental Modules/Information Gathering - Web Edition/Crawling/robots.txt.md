## robots.txt

Imagine you are a guest at a large house party. You are free to explore most rooms, but some doors are marked **“Private”** and you are expected not to enter them.
In the web world, **robots.txt** plays a similar role. It acts as an etiquette guide for web crawlers, indicating which parts of a website should or should not be accessed.

---

## What is robots.txt?

`robots.txt` is a simple text file placed in the **root directory** of a website, for example:

```
https://www.example.com/robots.txt
```

It follows the **Robots Exclusion Standard**, a set of guidelines that define how well-behaved crawlers should interact with a website. The file contains rules (called *directives*) that instruct bots which paths they may crawl and which they should avoid.

---

## How robots.txt Works

The rules in `robots.txt` are written for specific **user-agents**, which identify different crawlers. A basic example looks like this:

```
User-agent: *
Disallow: /private/
```

This tells **all bots** (`*` is a wildcard) not to crawl any URLs starting with `/private/`.

Other directives can:

* Allow access to specific paths
* Slow down crawling to reduce server load
* Point crawlers to XML sitemaps

---

## Structure of robots.txt

The file is plain text and consists of one or more **records**, separated by blank lines. Each record contains:

### 1. User-agent

Specifies which crawler the rules apply to.

Examples:

* `User-agent: *` → all bots
* `User-agent: Googlebot` → Google’s crawler only

### 2. Directives

Rules that apply to the specified user-agent.

Common directives include:

| Directive       | Description                             | Example                                    |
| --------------- | --------------------------------------- | ------------------------------------------ |
| **Disallow**    | Prevents crawling of specific paths     | `Disallow: /admin/`                        |
| **Allow**       | Explicitly permits crawling of a path   | `Allow: /public/`                          |
| **Crawl-delay** | Sets a delay between requests (seconds) | `Crawl-delay: 10`                          |
| **Sitemap**     | Points to an XML sitemap                | `Sitemap: https://example.com/sitemap.xml` |

---

## Why Respect robots.txt?

Although `robots.txt` is **not technically enforced**, most legitimate crawlers respect it. This matters for several reasons:

* **Server Protection**
  Prevents excessive crawling that could degrade performance or cause outages.

* **Reduced Indexing of Sensitive Areas**
  Helps keep admin panels, private directories, or staging paths out of search engines.

* **Legal and Ethical Considerations**
  Ignoring robots.txt may violate terms of service and, in some contexts, lead to legal issues.

---

## robots.txt in Web Reconnaissance

From a reconnaissance perspective, `robots.txt` is extremely valuable. Even when respecting its rules, it can reveal important intelligence:

### Key Recon Benefits

* **Hidden Directories**
  Disallowed paths often point directly to sensitive locations such as:

  * `/admin/`
  * `/backup/`
  * `/internal/`

* **Website Structure Clues**
  Allowed and disallowed paths provide insight into how the application is organised.

* **Security Awareness Indicators**
  Some sites include fake or monitored paths (honeypots) to detect malicious crawlers.

> A disallowed path is not a security control — it is often an **invitation to look more closely** during authorised testing.

---

## Example robots.txt Analysis

Example file:

```
User-agent: *
Disallow: /admin/
Disallow: /private/
Allow: /public/

User-agent: Googlebot
Crawl-delay: 10

Sitemap: https://www.example.com/sitemap.xml
```

### What We Can Infer

* `/admin/` likely hosts an administrative interface
* `/private/` may contain restricted or sensitive content
* `/public/` is explicitly intended for indexing
* Googlebot is rate-limited, suggesting concern about crawl load
* A sitemap exists, which may expose many valid URLs
