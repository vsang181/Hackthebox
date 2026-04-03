# Introduction to IDOR

Insecure Direct Object References (IDOR) is one of the most prevalent and impactful vulnerability classes in web security, appearing consistently in bug bounty programmes and real-world penetration tests across every type of web application. Unlike many vulnerabilities that require complex payloads or chained exploits, IDOR typically requires nothing more than changing a number in a URL.

***

## What IDOR Actually Is

The vulnerability has two required components that must both be present:

1. The application exposes a direct reference to an internal object (a file ID, user ID, database record ID, or filename)
2. The back-end does not verify that the requesting user is authorised to access that specific object

The reference itself is not the problem. Every application needs to identify its resources somehow. The problem is the absent or broken ownership check on the back-end. When a user requests `/download.php?file_id=123`, a secure application verifies that the authenticated user owns file 123 before returning it. A vulnerable application simply fetches file 123 and returns it to whoever asked.

```
Secure flow:
  Request: GET /download.php?file_id=123
  Back-end: SELECT * FROM files WHERE id=123 AND user_id=[session_user_id]
  Result: Returns file only if it belongs to the logged-in user

Vulnerable flow:
  Request: GET /download.php?file_id=123
  Back-end: SELECT * FROM files WHERE id=123
  Result: Returns file regardless of who is asking
```

Changing `file_id=123` to `file_id=124` exposes any other user's file with no additional effort.

***

## Why IDOR is So Common

Building a complete access control system is architecturally difficult and easy to get wrong in specific places even when the overall system is correctly designed. Several factors contribute to IDOR's prevalence:

- Developers focus on authentication (verifying who you are) and neglect authorisation (verifying what you can access)
- Access control logic is easy to implement globally for page-level access but harder to apply consistently at the object level within pages
- Front-end restrictions (only showing your own data) create a false sense of security that the back-end is also protected
- Automated scanners struggle to detect IDOR because the scanner cannot know which IDs belong to which user without application context
- Sequential integer IDs are ubiquitous in database design, making enumeration trivial once one valid ID is known

Facebook, Instagram, and Twitter have all had publicly disclosed IDOR vulnerabilities despite their dedicated security teams, which demonstrates that the vulnerability class persists even under rigorous security review.

***

## IDOR Vulnerability Types

### Information Disclosure IDOR

The most common form: reading data belonging to another user by manipulating an identifier.

```
/profile?user_id=1001        → your profile
/profile?user_id=1002        → another user's full profile data
/api/invoices/download/5543  → your invoice
/api/invoices/download/5544  → another user's invoice with payment details
```

### Modification IDOR

The same reference exposure applies to write operations. If the back-end does not check ownership on update or delete operations, any user can modify or destroy any other user's data:

```
POST /api/profile/update
Body: {"user_id": "1002", "email": "attacker@evil.com"}
→ Changes another user's email address if no ownership check exists
```

### Insecure Function Calls (Privilege Escalation)

Some applications expose admin-only API endpoints or function parameters in the front-end code but rely on the front-end to hide them from non-admin users. The back-end never independently verifies the caller's role:

```
POST /api/admin/change-role
Body: {"user_id": "1001", "role": "admin"}
→ If back-end does not verify the caller is an admin, any user can escalate
```

This IDOR variant leads directly to full application takeover since an attacker who grants themselves admin access can then access every user's data and every administrative function.

***

## The Access Control Gap

The core issue is that many applications implement two separate security layers that do not communicate:

```
Front-end layer:
  - Only renders your own profile data
  - Hides admin buttons from non-admin users
  - Only shows your own files in the file manager
  - Correctly scoped: you never see other users' data in the UI

Back-end layer (vulnerable):
  - Receives any user_id parameter and fetches that record
  - No comparison between session user and requested resource
  - Processes admin API calls regardless of caller's role
```

The front-end restrictions are entirely bypassed by sending a direct HTTP request with a modified parameter, either through Burp Suite, curl, or any HTTP client. The back-end processes the request with no awareness that the front-end was supposed to prevent it.

***

## Impact Severity

IDOR impact scales directly with what the exposed object contains and what operations are permitted:

| IDOR Type | Example | Maximum Impact |
|-----------|---------|---------------|
| Read on personal data | Profile, address, DOB | Privacy violation, PII disclosure |
| Read on financial data | Invoices, card numbers | Financial fraud enablement |
| Read on authentication data | Password reset tokens, session IDs | Account takeover |
| Write on user data | Email, password, address change | Account takeover |
| Write on role/permission data | Privilege escalation | Full application compromise |
| Delete on any object | File deletion, account removal | Data destruction, DoS |

The progression from information disclosure to full account takeover often requires only a single additional step: an IDOR that leaks a password reset token, or one that allows changing the email address on an account, converts immediately from a data exposure into complete ownership of that account.
