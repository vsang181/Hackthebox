# IDOR Prevention

Preventing IDOR requires two independent layers working together: a robust server-side access control system, and object references that cannot be predicted or enumerated. Neither layer alone is sufficient.

***

## Object-Level Access Control

The root cause of every IDOR covered in this module was the same: the back-end trusted values supplied by the client (uid in a cookie, role in a JSON body, uuid in a POST parameter) instead of deriving them from an authenticated server-side session. A properly designed access control system removes that trust entirely.

Role-Based Access Control (RBAC) is the standard pattern for this. Every object in the application is mapped to a set of required privileges, and every request is validated against the authenticated user's actual server-side role, not anything the client sends:

```javascript
// Secure: role is read from the back-end RBAC using the session token
// The client never touches the role value in this flow
match /api/profile/{userId} {
    allow read, write: if user.isAuth == true
    && (user.uid == userId || user.roles == 'admin');
}
```

Compare this to the vulnerable pattern seen throughout the module:

```
Vulnerable:  PUT /api/profile/1  body: { "role": "web_admin" }
             → back-end reads role FROM the request body

Secure:      PUT /api/profile/1  header: Authorization: Bearer <JWT>
             → back-end ignores role in body, reads it from the token's claims
               which were set at login time and signed server-side
```

The distinction is where authority lives. If the client can write it, the client can abuse it.

***

## Implementing RBAC Checks in Code

A per-endpoint access control check should be the first thing any sensitive function runs before touching a database:

```python
# Python / Flask example of proper object-level access control
from flask import request, abort
import jwt

def get_profile(requested_uid):
    # Derive identity from signed session token, not from request body
    token = request.headers.get("Authorization").split(" ")[1]
    claims = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
    
    requester_uid = claims["uid"]
    requester_role = claims["role"]  # from server-side DB at login, not client-supplied
    
    # Only allow if requesting own profile OR is admin
    if requester_uid != requested_uid and requester_role != "admin":
        abort(403)  # Access denied before any DB query runs
    
    return db.query("SELECT * FROM users WHERE uid = ?", requested_uid)
```

The check happens before the database query. The role value never comes from the request body or a cookie that the user could modify.

***

## Secure Object Referencing

Even with solid access control, using sequential integers like `uid=1, uid=2` makes enumeration trivial and makes it easy to accidentally miss an access check. Strong references eliminate the enumeration surface entirely.

Use UUID v4 for all externally exposed object references:

```php
// Generating a UUID v4 reference when creating a new resource
$uuid = sprintf('%04x%04x-%04x-%04x-%04x-%04x%04x%04x',
    mt_rand(0, 0xffff), mt_rand(0, 0xffff),
    mt_rand(0, 0xffff),
    mt_rand(0, 0x0fff) | 0x4000,
    mt_rand(0, 0x3fff) | 0x8000,
    mt_rand(0, 0xffff), mt_rand(0, 0xffff), mt_rand(0, 0xffff)
);

// Store mapping in DB: uuid → internal uid
$query = "INSERT INTO uuid_map (uuid, uid) VALUES ('$uuid', $uid)";

// Expose only the UUID externally, never the integer uid
echo "/profile/$uuid";
```

```php
// When a request comes in using the UUID
$uuid = $_REQUEST['uuid'];  // e.g. "89c9b29b-d19f-4515-b2dd-abb6e693eb20"

// Resolve internally, after the access control check passes
$query = "SELECT url FROM documents WHERE uuid = ?";
```

The external world only ever sees the UUID. Internal integer IDs never leave the server.

***

## Hashing: Server-Side Only

The module demonstrated how front-end hashing gives attackers the full algorithm. The rule is straightforward:

| Approach | Security |
|----------|---------|
| Hash calculated in JavaScript, sent to server | Attacker reads the JS, replicates the hash for any input |
| UUID generated server-side at object creation, stored in DB | No algorithm to reverse, no sequential pattern to enumerate |
| md5(base64(uid)) calculated server-side per request | Server knows the mapping; client only ever sees the output |

Never compute a reference value client-side that the server will later trust as authoritative. Compute it once at object creation time, store it, and serve it.

***

## Why UUID Alone is Not Enough

A common mistake is implementing UUIDs and considering the problem solved. The module explicitly addresses this:

```
Scenario: Strong UUIDs but broken access control

GET /api/profile/89c9b29b-d19f-4515-b2dd-abb6e693eb20
→ Back-end returns the profile with no session check
→ UUID is hard to guess, but if any other path leaks it
  (e.g. a user list endpoint, an email, a search result),
  it becomes exploitable again
```

UUID makes enumeration hard, not impossible. If the access control check is missing or broken, a UUID just raises the bar slightly rather than preventing the attack. The correct mental model is:

```
Secure Object Reference = Access Control (server-side RBAC) + Unpredictable Reference (UUID)
Both must be present. Either alone is insufficient.
```

***

## Prevention Checklist

A practical checklist for reviewing any API endpoint for IDOR exposure:

- Every GET/PUT/POST/DELETE endpoint validates the requester's identity from a server-side session token before touching any data
- Role values are never read from the request body, query string, or a client-modifiable cookie
- Integer sequential IDs are never exposed in URLs, API parameters, or response bodies
- All external object references are UUID v4 or cryptographically salted hashes generated at object creation
- All hashing or encoding logic lives in back-end code only, never in JavaScript delivered to the browser
- Access control checks are centralised (middleware or a shared function), not duplicated per endpoint, to prevent gaps
- GET endpoints have the same access control rigour as write endpoints, since GET responses often leak the data needed to exploit write endpoints
