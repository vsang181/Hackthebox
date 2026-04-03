# Web Attacks: Introduction

Web applications represent the largest and most accessible attack surface for most organisations today. Whether customer-facing, internal, or API-based, they all share the same fundamental weakness: they accept user-controlled input and process it in ways that can be subverted. This module focuses on three vulnerabilities that appear across virtually every web application stack, regardless of language, framework, or hosting environment.

***

## HTTP Verb Tampering

The HTTP specification defines a range of methods beyond the familiar GET and POST, including PUT, DELETE, HEAD, OPTIONS, PATCH, and TRACE. Web servers and application frameworks often handle these inconsistently, applying security controls to some methods while leaving others unrestricted.

The attack exploits this gap directly. If an admin panel returns 403 on a GET request but the server accepts the same path via POST or HEAD without checking authorisation, the attacker accesses restricted functionality simply by changing the request method. The server's access control list checked the verb it expected but never considered the alternative.

Key impact scenarios:
- Bypassing authentication on protected endpoints
- Circumventing security filters that only inspect specific HTTP methods
- Accessing administrative functionality without valid credentials
- Exploiting configuration mismatches between web server and application framework

***

## Insecure Direct Object References (IDOR)

IDOR is consistently one of the most commonly reported vulnerabilities in bug bounty programmes because it requires no special tooling, no payload crafting, and no technical sophistication to exploit. The application hands the attacker a reference to an object, and the attacker simply changes it to reference a different object.

The root cause is a missing or incorrectly implemented authorisation check on the back-end. The application retrieves whichever record matches the supplied identifier without verifying that the requesting user owns or is permitted to access that record.

```
/api/user/profile?id=1001    → your profile
/api/user/profile?id=1002    → another user's profile (if IDOR present)
/download?file=invoice_1001.pdf  → your invoice
/download?file=invoice_1002.pdf  → another user's invoice
```

The identifier does not need to be a sequential integer. Base64 encoded IDs, hashed filenames, and GUIDs can all be vulnerable if the underlying authorisation check is absent, since the encoding only obscures the reference without protecting it.

***

## XML External Entity (XXE) Injection

XXE targets the XML parsing layer rather than the application logic. The XML specification includes a feature called external entities, which allows an XML document to reference and include content from an external source. When a vulnerable parser processes user-supplied XML containing a malicious entity declaration, it fetches and includes the referenced content, which may be a local file, an internal network resource, or a cloud metadata endpoint.

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<root><data>&xxe;</data></root>
```

The parser reads `/etc/passwd` and substitutes its contents at `&xxe;`. The application returns that content in its response, effectively giving the attacker a file read primitive without any code execution required.

The impact escalates beyond simple file disclosure:
- Reading application source code and configuration files containing credentials
- Probing internal network services that are not externally accessible (SSRF)
- Accessing cloud instance metadata endpoints for IAM credentials (AWS, GCP, Azure)
- Denial of service via recursive entity expansion (Billion Laughs attack)
- Credential theft leading to full server compromise and remote code execution

***

## Why These Three Together

These vulnerabilities are grouped because they represent three fundamentally different failure modes that independently lead to serious compromise:

| Attack | Root Cause | Primary Impact |
|--------|-----------|---------------|
| HTTP Verb Tampering | Incomplete access control across HTTP methods | Authentication and authorisation bypass |
| IDOR | Missing object-level authorisation checks | Unauthorised data access and modification |
| XXE | Unsafe XML parser configuration | File disclosure, SSRF, credential theft |

Each can be exploited individually, but they also chain effectively. XXE provides source code access that reveals IDOR-vulnerable endpoints. Those endpoints expose user data and session tokens. HTTP verb tampering on an admin function provides the final escalation. Understanding all three as a connected attack surface rather than isolated issues reflects how real-world compromise chains are actually constructed.
