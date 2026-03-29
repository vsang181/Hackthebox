# Handling SQLMap Errors

When SQLMap fails to detect or exploit an injection point, the problem is almost always in how the request is constructed or how the tool is interpreting the target's responses. Four debugging tools give you progressively deeper visibility into what is happening.

***

## Debugging Tools in Order of Use

Start with `--parse-errors` for quick DBMS-level feedback, then escalate to `-v 6` or `-t` if the problem is not immediately obvious, and use `--proxy` when you need to interact with requests manually in Burp.

### 1. --parse-errors (First Stop)

Extracts and displays raw database error messages from the response:

```bash
sqlmap -u "http://target.com/vuln.php?id=1" --parse-errors --batch
```

Output when working:
```
[WARNING] parsed DBMS error message: 'SQLSTATE[42000]: Syntax error or 
access violation: 1064 You have an error in your SQL syntax; check the 
manual that corresponds to your MySQL server version for the right syntax 
to use near '))"',),)((' at line 1'
```

This single flag tells you the database is receiving the payload and responding with errors, confirming the injection point is active. The error content also reveals the DBMS version and exact query context, helping you understand what quote or bracket structure the original query uses.

### 2. -t (Traffic File)

Saves every request and response to a file for offline inspection:

```bash
sqlmap -u "http://target.com/vuln.php?id=1" --batch -t /tmp/traffic.txt
cat /tmp/traffic.txt
```

The traffic file shows the full HTTP exchange for every request:

```
HTTP request [#1]:
GET /?id=1 HTTP/1.1
Host: www.example.com
User-agent: sqlmap/1.4.9

HTTP response [#1] (200 OK):
Content-Type: text/html; charset=UTF-8
...full response body...
```

Use `-t` when you need to audit exactly what payloads were sent, verify that cookies and headers are being transmitted correctly, or compare request sequences to understand why a particular technique is failing.

### 3. -v (Verbosity Levels)

Controls how much output appears in the terminal during the run:

| Level | Output Includes |
|-------|----------------|
| 0 | Errors only |
| 1 | Basic info (default) |
| 2 | Debug messages |
| 3 | Payloads sent |
| 4 | HTTP request headers |
| 5 | HTTP response headers |
| 6 | Full HTTP requests and responses, real-time |

```bash
# Most useful debugging level
sqlmap -u "http://target.com/vuln.php?id=1" -v 6 --batch
```

Level 6 prints every request and response body to the terminal as they happen, combining the functionality of `--parse-errors` and `-t` but in real time. It is the most complete view of what SQLMap is doing but generates significant terminal output on a full scan. Use it when a specific parameter is not being detected and you need to see exactly what payload the tool is sending and what the application is returning.

### 4. --proxy (Route Through Burp)

Redirects all SQLMap traffic through a proxy:

```bash
# Route through Burp Suite
sqlmap -u "http://target.com/vuln.php?id=1" --proxy="http://127.0.0.1:8080" --batch

# With proxy authentication
sqlmap -u "http://target.com/vuln.php?id=1" \
  --proxy="http://127.0.0.1:8080" \
  --proxy-cred="user:pass" --batch
```

This is the most powerful debugging approach because Burp's HTTP history captures every request SQLMap sends with full detail, and you can then manually repeat, modify, or forward individual requests. It also lets you use Burp's Comparer tool to see exactly what differs between a TRUE and FALSE response that SQLMap is using for boolean-based detection.

For this to work, Burp must be running with its proxy listener active on `127.0.0.1:8080`. If the target uses HTTPS, import Burp's CA certificate into the OS trust store or use `--proxy-cert` to avoid TLS errors interrupting the scan.

***

## Practical Debugging Workflow

```
Problem: SQLMap not finding injection

Step 1: Add --parse-errors
  → DBMS errors visible? Injection point confirmed, payload syntax issue
  → No errors? Request may not be reaching the DB at all

Step 2: Add -v 3
  → Check payloads being sent match expected injection format
  → Confirm correct parameter is being targeted

Step 3: Add -t /tmp/traffic.txt
  → Review full request/response pairs offline
  → Verify cookies, headers, and POST body are correct

Step 4: Add --proxy=http://127.0.0.1:8080
  → Inspect requests in Burp HTTP history
  → Manually replay requests with modified payloads
  → Use Burp Repeater to confirm injection point works manually
  → Feed confirmed working request back to SQLMap via -r req.txt
```

The combination of `--parse-errors` and `--proxy` covers the majority of real-world debugging scenarios. Parse errors tell you the database is responding, and the proxy lets you manually verify the request is formed correctly before trusting SQLMap's automated detection result.
