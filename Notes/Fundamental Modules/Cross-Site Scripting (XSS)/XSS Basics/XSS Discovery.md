# XSS Discovery

Finding XSS vulnerabilities requires a layered approach: automated tools for broad coverage, manual payload testing for confirmation, and code review for the most reliable but time-intensive results.

***

## Automated Discovery

Web application scanners test for XSS by injecting payloads into every discovered input and comparing the rendered response to see if the payload survived unmodified. Two scan modes exist:

- **Passive scan**: Reviews client-side JavaScript for dangerous source-to-sink data flows without sending injection payloads, primarily finding DOM-based vulnerabilities
- **Active scan**: Sends crafted payloads to every input parameter and analyses responses for reflected content

| Tool | Type | Cost | Notes |
|------|------|------|-------|
| Burp Suite Pro | Active + Passive | Paid | Highest accuracy, handles complex bypass scenarios |
| OWASP ZAP | Active + Passive | Free | Good coverage, slightly lower accuracy than Burp Pro |
| Nessus | Active | Paid | Better suited for infrastructure than web app testing |
| XSStrike | Active | Free | Context-aware payload generation, WAF detection |
| BruteXSS | Active | Free | Brute force payload approach |
| XSSer | Active | Free | Large payload library, multiple injection vectors |

Automated tools always require manual verification. A payload appearing in the source does not guarantee execution, and many false positives occur because the tool sees reflection without confirming whether the browser would execute the code.

### XSStrike Usage

```bash
git clone https://github.com/s0md3v/XSStrike.git
cd XSStrike
pip install -r requirements.txt

# Test a GET parameter
python xsstrike.py -u "http://target.com/index.php?task=test"

# Test POST parameters
python xsstrike.py -u "http://target.com/login.php" --data "user=test&pass=test"

# Crawl the entire application
python xsstrike.py -u "http://target.com/" --crawl

# Skip DOM vulnerability checks (faster)
python xsstrike.py -u "http://target.com/?id=1" --skip-dom
```

XSStrike's key advantage over basic payload lists is context-aware payload generation: it analyses the reflection context (inside a tag, inside an attribute, inside a script block) and generates payloads specifically tailored to that context rather than blindly trying generic payloads.

***

## Manual Payload Testing

Manual testing works by submitting payloads from a curated list and observing whether execution occurs. Key payload sources:

- PayloadsAllTheThings XSS section (GitHub)
- Payload-Box XSS payload list (GitHub)
- PortSwigger XSS cheat sheet (most comprehensive for context-specific bypass)

### Payloads by Injection Context

The injection context determines which payload structure will work. A payload that succeeds in a `<div>` body will fail inside an HTML attribute or a JavaScript string:

```html
<!-- HTML body context: direct tag injection -->
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

<!-- HTML attribute context: break out of attribute first -->
" onmouseover="alert(1)
" autofocus onfocus="alert(1)

<!-- JavaScript string context: escape the string first -->
'; alert(1)//
\'; alert(1)//

<!-- URL parameter context -->
javascript:alert(1)

<!-- innerHTML sink: script tags blocked, use event handlers -->
<img src='' onerror=alert(1)>
<details open ontoggle=alert(1)>
```

### Injection Points Beyond Input Fields

XSS is not limited to visible form fields. Any location where user-controlled data reaches the HTML output is a potential injection point:

- HTTP headers: `User-Agent`, `Referer`, `X-Forwarded-For` when displayed in admin panels or logs
- Cookie values when reflected in page content
- URL path segments when included in breadcrumb navigation
- HTTP response error messages containing request data
- File upload filenames when displayed after upload
- JSON API responses rendered as HTML

***

## Code Review (Most Reliable Method)

Manual code review traces the data flow from user input to output without needing to trigger execution. For front-end JavaScript (DOM XSS), the pattern to find is:

```
[Attacker-controlled source] → [No sanitisation] → [Dangerous sink]
```

**Sources to search for in JavaScript:**
```javascript
document.URL
document.location
document.location.href
document.location.hash
window.location.search
document.referrer
document.cookie
```

**Sinks to search for:**
```javascript
innerHTML =
outerHTML =
document.write(
eval(
setTimeout(
setInterval(
$.html(
$.append(
```

For back-end code (stored/reflected XSS), the pattern is unsanitised output:

```php
// Vulnerable: raw output
echo $_GET['input'];
echo "<p>" . $userInput . "</p>";

// Vulnerable: only checking for script tags (bypassable)
echo str_replace('<script>', '', $input);

// Secure: proper encoding
echo htmlspecialchars($input, ENT_QUOTES, 'UTF-8');
```

***

## Discovery Workflow

```
1. Map all input points
   → Form fields, URL parameters, headers, cookies, file names

2. Run automated scanner (XSStrike or Burp Active Scan)
   → Quick broad coverage, identify candidates

3. Manual verification of flagged parameters
   → Confirm execution, not just reflection
   → Test with alert(window.origin) to confirm origin

4. Manual payload testing for missed parameters
   → Context-specific payloads based on where input lands in source

5. Code review for any accessible source
   → Search for dangerous sinks receiving unsanitised input
   → Particularly valuable for DOM XSS which scanners miss
```

The reason many payloads from generic lists fail even on genuinely vulnerable pages is that those payloads were written for specific injection contexts or to bypass specific filters. A `<script>alert(1)</script>` payload works perfectly in an HTML body context but does nothing inside a JavaScript string, where `'; alert(1)//` is needed instead. Understanding the context before selecting a payload is more efficient than blind iteration through a 500-entry list.
