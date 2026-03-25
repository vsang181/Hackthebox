# Page Fuzzing

Once a directory is discovered, the next step is finding the actual files and pages within it. This requires two things: knowing what file extension the application uses, and then fuzzing for filenames with that extension.

***

## Step 1: Extension Fuzzing

Rather than guessing the extension from the server type in response headers, fuzzing for it directly is faster and more reliable. The technique uses `index` as the known base filename (since virtually every web application has an index file) and fuzzes only the extension:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
     -u http://SERVER_IP:PORT/blog/indexFUZZ
```

Note that the wordlist `web-extensions.txt` already includes the dot before each extension (`.php`, `.html`, `.aspx`, etc.), so the URL is written as `indexFUZZ` rather than `index.FUZZ`. The dot is part of the payload.

Example output:

```
.php     [Status: 200, Size: 0, Words: 1, Lines: 1]
.phps    [Status: 403, Size: 283, Words: 20, Lines: 10]
```

A 200 response on `.php` confirms the application runs PHP. A 403 on `.phps` confirms that extension is recognised by the server but access is restricted, which is still useful information. Now you know the extension to use for the next step.

***

## Step 2: Page Fuzzing

With the extension confirmed as `.php`, fuzz for page names within the directory by appending `.php` statically to the `FUZZ` keyword:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://SERVER_IP:PORT/blog/FUZZ.php
```

Example output:

```
index      [Status: 200, Size: 0, Words: 1, Lines: 1]
home       [Status: 200, Size: 465, Words: 42, Lines: 15]
```

Both return 200, but the response size tells you which is worth investigating. A size of 0 means an empty page. A page with 465 bytes has actual content and is the one to visit manually.

***

## The Two-Step Workflow

This two-step approach is the standard pattern for thorough page discovery:

```
1. Discover directories  →  ffuf with FUZZ in the path
2. Identify extension    →  ffuf with indexFUZZ
3. Fuzz for pages        →  ffuf with FUZZ.php (or relevant extension)
```

Each step feeds into the next. Skipping extension fuzzing and assuming `.php` because the server is Apache works most of the time, but applications built on Apache occasionally serve `.html`, `.py`, or other file types. Confirming the extension first eliminates guesswork.

***

## Common Extension to Server Mapping

While extension fuzzing is more reliable, this table is a useful starting reference when you need a quick guess:

| Server | Likely Extensions |
|--------|-----------------|
| Apache | `.php`, `.html` |
| Nginx | `.php`, `.html` |
| IIS | `.asp`, `.aspx` |
| Tomcat | `.jsp`, `.do` |
| Node.js | No extension or `.js` |
| Python | `.py`, no extension |

***

## Handling Empty Pages

An empty page (size 0) is not necessarily useless. It could mean:

- The file exists but only outputs content under specific conditions (e.g., requires a POST request, a valid session, or a specific parameter)
- It is a stub file left by a developer
- It processes a request and redirects without a body

Always note empty pages in your enumeration log and revisit them later when you have more context about how the application works. A page that appears empty during unauthenticated directory fuzzing may return substantial content once you have valid credentials or the correct parameter values.
