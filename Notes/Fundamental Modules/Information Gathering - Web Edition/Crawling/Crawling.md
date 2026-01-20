## Crawling

Crawling, also known as **spidering**, is the automated process of systematically browsing a website by following links from one page to another. Web crawlers are automated agents (bots) that traverse web applications using predefined rules, collecting information for indexing, analysis, or reconnaissance purposes.

In web reconnaissance, crawling is used to **map application structure**, **discover hidden or forgotten content**, and **extract data** that may not be immediately visible from the main entry points.

---

## How Web Crawlers Work

At a high level, a crawler operates using a simple but powerful loop:

1. Start with a **seed URL** (usually the homepage).
2. Fetch the page and parse its content.
3. Extract all discoverable links.
4. Add new links to a queue.
5. Visit each queued link and repeat the process.

This continues until the crawler reaches defined limits such as depth, scope, or time.

### Example Crawl Flow

```
Homepage
├── link1
├── link2
└── link3
```

Visiting `link1` reveals additional links:

```
link1 Page
├── Homepage
├── link2
├── link4
└── link5
```

The crawler continues iterating through newly discovered links, building an increasingly complete picture of the application.

> Crawling differs from fuzzing: crawling follows **existing, discoverable links**, whereas fuzzing **guesses** potential paths.

---

## Crawling Strategies

Crawlers generally follow one of two traversal strategies.

### Breadth-First Crawling (BFS)

Breadth-first crawling explores all links at the current depth before moving deeper.

<img width="2613" height="1632" alt="ig_crawling_1" src="https://github.com/user-attachments/assets/ad2f6d20-f536-43fa-b264-e69e90aeacea" />

* Prioritises **coverage**
* Useful for understanding overall site structure
* Ideal for large applications and content mapping

```
Seed
├── Page 1
│   ├── Page 2
│   └── Page 3
```

---

### Depth-First Crawling (DFS)

Depth-first crawling follows a single path as deep as possible before backtracking.

<img width="2613" height="1632" alt="ig_crawling_1" src="https://github.com/user-attachments/assets/2b8251f7-e097-4f73-a1cd-f2e9904953b0" />

* Prioritises **depth**
* Useful for reaching deeply nested functionality
* Can quickly uncover hidden workflows

```
Seed → Page 1 → Page 2 → Page 3
                     ├── Page 4
                     └── Page 5
```

The chosen strategy depends on reconnaissance goals, scope, and time constraints.

---

## Data Extracted During Crawling

Well-configured crawlers can extract a wide range of valuable information:

### Links

* **Internal links** reveal application structure and hidden pages
* **External links** expose third-party integrations and dependencies

### Comments

* User or developer comments may leak:

  * Internal system details
  * File paths
  * Software versions
  * Operational assumptions

### Metadata

Includes:

* Page titles
* Descriptions
* Keywords
* Authors
* Timestamps

Metadata often reveals application purpose, functionality, or outdated components.

### Sensitive Files

Crawlers can be tuned to identify exposed files such as:

* Backup files (`.bak`, `.old`, `.zip`)
* Configuration files (`web.config`, `settings.php`, `.env`)
* Logs (`access.log`, `error.log`)
* Temporary or test artefacts

These files frequently contain:

* Credentials
* API keys
* Database connection strings
* Source code fragments

---

## The Importance of Context

Individual findings rarely tell the full story on their own. The real value of crawling emerges when **multiple data points are correlated**.

Examples:

* A comment referencing a “file server” may seem harmless
* Multiple URLs pointing to `/files/`
* Manual inspection reveals directory listing enabled
* Backup archives and internal documents are exposed

Another example:

* Metadata references an outdated framework version
* Crawled JavaScript files confirm the same version
* Public exploit exists for that version

In isolation, each finding is low risk. Together, they form a **high-confidence attack path**.
