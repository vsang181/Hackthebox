# Basic HTTP Authentication

Basic Auth is one of the oldest web authentication mechanisms and remains in use on internal tools, router admin panels, and legacy applications. Its simplicity is also its weakness: credentials are only Base64-encoded, not encrypted, making interception trivial on unencrypted connections and brute forcing straightforward.

***

## How Basic Auth Works

The exchange follows a fixed pattern on every request:

1. Client requests a protected resource
2. Server responds with `401 Unauthorized` and a `WWW-Authenticate: Basic realm="..."` header
3. Browser prompts the user for credentials
4. Browser concatenates them as `username:password`, Base64-encodes the result, and sends it in every subsequent request

```http
GET /protected_resource HTTP/1.1
Host: www.example.com
Authorization: Basic YWxpY2U6c2VjcmV0MTIz
```

Decoding `YWxpY2U6c2VjcmV0MTIz` reveals `alice:secret123` in plain text. Base64 is an encoding scheme, not encryption. Anyone intercepting the traffic can decode credentials instantly with:

```bash
echo "YWxpY2U6c2VjcmV0MTIz" | base64 -d
# alice:secret123
```

This is why Basic Auth should only ever be deployed over HTTPS, though many internal services skip this requirement.

***

## Exploiting Basic Auth with Hydra

The `http-get` module handles Basic Auth natively. Hydra constructs the `Authorization` header automatically for each credential pair, so no manual encoding is needed.

```bash
# Download the wordlist
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/master/Passwords/Common-Credentials/2023-200_most_used_passwords.txt

# Run the attack
hydra -l basic-auth-user \
      -P 2023-200_most_used_passwords.txt \
      127.0.0.1 \
      http-get / \
      -s 81
```

Flag breakdown:

| Flag | Purpose |
|------|---------|
| `-l basic-auth-user` | Single known username, skips username enumeration entirely |
| `-P 2023-200_most_used_passwords.txt` | Password wordlist, 200 entries |
| `127.0.0.1` | Target IP |
| `http-get /` | Module and path; `/` is the protected resource |
| `-s 81` | Non-standard port override |

The scan completes in roughly one second against a 200-entry list because Basic Auth over HTTP is extremely fast to brute force. There is no session handling, no CSRF token, no JavaScript challenge, just a header check on every request.

***

## Knowing the Username Matters

The module highlights a subtle but important point: knowing the username in advance halves the attack surface. Without a known username you would need both `-L usernames.txt` and `-P passwords.txt`, multiplying the total attempts by the number of usernames. A 200-password list against 17 common usernames becomes 3,400 attempts. With the username confirmed as `basic-auth-user`, it stays at 200.

Username discovery before brute forcing is worth the upfront time investment. Common sources include:

- Error messages that distinguish `user not found` from `wrong password`
- OSINT on company employees via LinkedIn
- Email format inference (`firstname.lastname@company.com`)
- Default credentials documentation for the specific service

***

## Basic Auth vs Form-Based Auth in Hydra

These two are the most commonly confused because they both target HTTP, but the Hydra module and syntax differ significantly:

| Aspect | Basic Auth | Form-Based Auth |
|--------|-----------|----------------|
| Hydra module | `http-get` | `http-post-form` |
| Credentials location | `Authorization` header | POST body |
| Hydra handles encoding | Yes, automatic | No, you define the parameters |
| Failure detection | HTTP 401 | Custom string match required |
| Server-side session | No | Often yes (cookies, tokens) |

Basic Auth is simpler to attack precisely because Hydra handles everything automatically. Form-based auth requires you to inspect the login form, identify parameter names, and define a failure or success string, which is covered in the web form fuzzing sections.

***

## Defences Against Basic Auth Brute Forcing

Basic Auth has no built-in lockout or rate limiting. All protection must come from the server or a layer in front of it:

- Place a reverse proxy (nginx, Caddy) in front that enforces rate limiting and IP-based lockouts
- Use `fail2ban` to monitor auth logs and ban IPs after repeated 401 responses
- Require HTTPS so credentials cannot be intercepted in transit
- Migrate to a more robust authentication mechanism (form-based auth with CSRF tokens, OAuth, or SSO) for anything internet-facing
- Restrict access to authorised IP ranges at the network level where the resource is internal-only
