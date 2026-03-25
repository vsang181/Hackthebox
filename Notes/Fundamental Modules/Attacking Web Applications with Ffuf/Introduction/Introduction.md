# Attacking Web Applications with Ffuf: Introduction

Ffuf is one of the most reliable and widely used web fuzzing tools available today. Its speed, flexibility, and clean output make it the preferred CLI fuzzer for directory brute-forcing, parameter discovery, virtual host enumeration, and file extension scanning during web application penetration tests.

***

## What This Module Covers

The module works through five core fuzzing techniques that together give you a comprehensive picture of a target web application's attack surface:

- **Directory fuzzing**: Discovering hidden or unlisted directories by brute-forcing path names against the web root
- **File and extension fuzzing**: Finding files that exist on the server by testing known filenames and extensions like `.php`, `.bak`, `.txt`, and `.html`
- **Virtual host (vhost) identification**: Enumerating subdomains and virtual hosts that may be hosted on the same IP but respond differently based on the `Host` header
- **PHP parameter fuzzing**: Discovering hidden GET and POST parameters that the application accepts but does not expose in its public interface
- **Parameter value fuzzing**: Once a parameter is identified, testing its accepted values to uncover logic flaws, injection points, or access control bypasses

***

## The Core Concept

The fundamental principle behind all ffuf usage is straightforward. You place the `FUZZ` keyword at the injection point in your request, provide a wordlist, and ffuf iterates through every line of that wordlist substituting it into the `FUZZ` position and sending the request. A 200 response means the resource exists. A 301 or 302 means it exists but redirects elsewhere. A 404 means it does not exist.

```bash
ffuf -u http://TARGET/FUZZ -w /path/to/wordlist.txt
```

The challenge in practice is not running the tool but interpreting and filtering the output cleanly. Many applications return custom error pages with 200 status codes, meaning you cannot rely on status codes alone and need to use size-based or content-based filtering to separate genuine hits from noise.

***

## Why Fuzzing Matters

You cannot test what you cannot find. Before any vulnerability assessment can begin in earnest, you need to map the full scope of what the application exposes. Developers frequently:

- Leave administrative panels at predictable but unlisted paths like `/admin/`, `/dashboard/`, or `/manager/`
- Deploy backup or test files with revealing extensions like `.bak`, `.old`, or `.swp`
- Accept undocumented parameters that bypass authentication or trigger debug behaviour
- Run internal virtual hosts on the same IP that have weaker security than the primary domain

Ffuf automates the discovery phase that would take hours manually, compressing it into minutes depending on wordlist size and thread count. The skills built in this module directly feed into every other web application testing technique: SQL injection, command injection, authentication bypass, and file upload attacks all start with finding the entry points that automated scanning alone might miss.
