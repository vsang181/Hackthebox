# Directory Fuzzing with Ffuf

Directory fuzzing is the most common starting point for web application enumeration. The goal is to discover paths on the server that are not publicly linked by sending a wordlist of common directory names and recording which ones return a meaningful response.

***

## Basic Command Structure

Ffuf uses the `FUZZ` keyword as a placeholder that gets replaced by each line of the wordlist. The wordlist is assigned to this keyword using a colon separator in the `-w` flag:

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
     -u http://SERVER_IP:PORT/FUZZ
```

This tells ffuf to take every line from the wordlist, substitute it into the `FUZZ` position in the URL, send the request, and report back results that match its default status code list (`200, 204, 301, 302, 307, 401, 403`).

***

## Understanding the Output

```
blog    [Status: 301, Size: 326, Words: 20, Lines: 10]
```

Each result line shows:

| Field | Meaning |
|-------|---------|
| Payload | The wordlist entry that produced this result |
| Status | HTTP response code |
| Size | Response body size in bytes |
| Words | Word count in the response |
| Lines | Line count in the response |

A 301 response means the directory exists and the server is redirecting to it (usually adding a trailing slash). The directory is accessible even though there may not be an index page at the root of it.

***

## Key Flags Reference

| Flag | Purpose |
|------|---------|
| `-w` | Wordlist path with optional keyword assignment (`wordlist:KEYWORD`) |
| `-u` | Target URL with `FUZZ` marker |
| `-t` | Thread count (default: 40) |
| `-mc` | Match specific status codes |
| `-fc` | Filter out specific status codes |
| `-fs` | Filter by response size (removes false positives) |
| `-ic` | Ignore wordlist comments (removes copyright lines) |
| `-c` | Coloured output |
| `-v` | Verbose output |
| `-o` | Save results to a file |
| `-recursion` | Recursively fuzz discovered directories |
| `-recursion-depth` | Limit recursion depth |

***

## Speed Considerations

Ffuf defaults to 40 threads and typically achieves thousands of requests per second depending on network latency. A full run against `directory-list-2.3-small.txt` (~87k entries) completes in under 10 seconds in optimal conditions.

You can increase threads with `-t 200` for faster results, but this comes with real risks:

- Against remote targets, high thread counts can trigger rate limiting or IDS alerts
- Aggressive scanning can cause unintended denial of service on fragile applications
- It can saturate your own network connection

For lab environments, higher thread counts are fine. For real engagements, keep threads at a reasonable level and check your rules of engagement before increasing them.

***

## Interpreting a Hit

When ffuf finds a directory like `/blog` returning a 301, visiting it manually is always worth doing even if the page appears empty. An empty index does not mean the directory has nothing in it. It means:

- The server allows directory listing to be disabled
- There may be files inside the directory that are not linked
- There may be subdirectories that require another fuzzing pass

This is exactly why the next logical step after directory fuzzing is file and extension fuzzing within each discovered directory. A directory that appears empty at the surface level often contains backup files, configuration files, or hidden pages when you fuzz one level deeper.

***

## Installation

Ffuf is pre-installed on PwnBox and most penetration testing distributions. To install on your own machine:

```bash
# Via apt
apt install ffuf -y

# From source (latest version)
git clone https://github.com/ffuf/ffuf.git
cd ffuf
go build .
```
