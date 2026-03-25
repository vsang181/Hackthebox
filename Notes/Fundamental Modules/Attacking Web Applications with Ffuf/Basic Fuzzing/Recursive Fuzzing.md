# Recursive Fuzzing

Manual multi-step fuzzing works for simple targets, but when an application has multiple directories each containing subdirectories and files, running separate scans for each level becomes impractical. Recursive fuzzing automates this by queuing a new scan automatically whenever a directory is discovered.

***

## How Recursion Works in Ffuf

When ffuf discovers a directory during a scan, it adds a new job to its internal queue targeting that directory. It then continues fuzzing within each discovered directory using the same wordlist and settings. This continues until no new directories are found or the specified depth limit is reached.

Without a depth limit, recursion on a deeply nested application can run for a very long time. Setting `-recursion-depth 1` limits the scan to:

- The web root and everything directly under it
- One level of subdirectories within anything discovered at the root

Anything deeper than that is not scanned automatically, giving you control over scope and runtime.

***

## The Recursive Scan Command

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://SERVER_IP:PORT/FUZZ \
     -recursion \
     -recursion-depth 1 \
     -e .php \
     -v
```

Flag breakdown:

| Flag | Purpose |
|------|---------|
| `-recursion` | Enables recursive scanning |
| `-recursion-depth 1` | Limits recursion to one level of subdirectories |
| `-e .php` | Appends `.php` to every wordlist entry in addition to testing the bare name |
| `-v` | Outputs the full URL for each result, making it clear which file belongs to which directory |

The `-e .php` flag effectively doubles the wordlist by testing each entry twice: once as a bare directory name and once with the `.php` extension appended. This is why recursive scans take significantly longer and send far more requests than a single-level scan.

***

## Reading Recursive Output

```
[Status: 301] | URL | http://SERVER_IP:PORT/blog --> http://SERVER_IP:PORT/blog/
[INFO] Adding a new job to the queue: http://SERVER_IP:PORT/blog/FUZZ
[Status: 200] | URL | http://SERVER_IP:PORT/blog/index.php
[Status: 200] | URL | http://SERVER_IP:PORT/forum/FUZZ
```

The `[INFO]` lines tell you when ffuf has discovered a directory and added it to the scan queue. This gives you real-time visibility into the scan tree as it builds. The `-v` flag is essential here because without full URLs you cannot tell whether `index.php` belongs to `/blog/`, `/forum/`, or the web root.

***

## Recursive vs Manual Multi-Step Fuzzing

| Approach | Speed | Coverage | Control |
|---------|-------|---------|---------|
| Manual (directory then page fuzzing) | Slower workflow | Targeted | Full control over each step |
| Recursive (`-recursion`) | Single command | Broader, automated | Less granular |
| Recursive with depth limit | Balanced | Root + one level | Prevents runaway scans |

***

## Practical Recommendations

For most assessments, start with a depth-1 recursive scan to get a broad overview of the application structure. Once you have the results, identify the most interesting directories (admin panels, API paths, upload directories) and run focused manual scans against those specifically with larger wordlists or different extensions.

Avoid running recursive scans with unlimited depth (`-recursion` without `-recursion-depth`) against production targets. A deeply nested application with large wordlists at each level can generate tens of thousands of requests and may trigger rate limiting, IDS alerts, or in extreme cases cause performance issues on the target server. Scoping your recursion keeps your testing controlled and professional.
