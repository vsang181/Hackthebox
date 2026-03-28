# Mitigating SQL Injection

No single defence eliminates SQLi risk entirely. The most resilient applications layer multiple controls so that bypassing one does not immediately lead to exploitation. Each method addresses a different point in the attack chain.

***

## Parameterised Queries (Primary Fix)

This is the only structural fix. Everything else reduces impact or adds friction; parameterised queries eliminate the injection vector entirely at the code level.

```php
// Vulnerable: user input concatenated directly into query string
$query = "SELECT * FROM logins WHERE username='" . $username . "' AND password='" . $password . "'";

// Secure: placeholders sent separately from data
$query = "SELECT * FROM logins WHERE username=? AND password=?";
$stmt = mysqli_prepare($conn, $query);
mysqli_stmt_bind_param($stmt, 'ss', $username, $password);
mysqli_stmt_execute($stmt);
```

The database engine receives the query structure first, then the parameter values separately. It treats the values as pure data regardless of their content. An attacker who injects `admin'--` into the username field sees it stored literally as the string `admin'--` rather than having the quote interpreted as SQL syntax. The `'ss'` in `bind_param` declares both parameters as strings, enforcing type safety.

***

## Input Sanitisation

Escaping special characters prevents them from being interpreted as SQL syntax. This is a secondary defence because escape functions can sometimes be bypassed with encoding tricks or unusual character sets:

```php
// Escapes ', ", \, NULL and other special characters
$username = mysqli_real_escape_string($conn, $_POST['username']);
$password = mysqli_real_escape_string($conn, $_POST['password']);
```

Sanitisation is database-specific: `mysqli_real_escape_string()` for MySQL, `pg_escape_string()` for PostgreSQL. Using the wrong escape function for the wrong database is a common mistake that leaves applications exposed.

***

## Input Validation

Validates that input conforms to an expected format before it ever reaches the query. Most effective when the input has a well-defined character set:

```php
// Port codes only contain letters and spaces
$pattern = "/^[A-Za-z\s]+$/";
if (!preg_match($pattern, $_GET['port_code'])) {
    die("Invalid input!");
}
```

The regex `^[A-Za-z\s]+$` rejects any input containing digits, quotes, semicolons, hyphens, or other SQL-relevant characters. An injection payload like `'; SELECT 1,2,3,4--` fails immediately at validation before reaching the database layer. Validation is context-dependent: an email field, numeric ID, or date field each has its own allowable pattern.

***

## Least Privilege Database Accounts

Limits the blast radius when injection does occur. A compromised restricted account cannot access other tables, other databases, or perform file operations:

```sql
-- Create a restricted application user
CREATE USER 'reader'@'localhost';
GRANT SELECT ON ilfreight.ports TO 'reader'@'localhost' IDENTIFIED BY 'StrongPassword!';
```

The `reader` account can only SELECT from the `ports` table in `ilfreight`. Attempting to read `credentials`, query `information_schema` for other databases, or use `LOAD_FILE()` all fail with permission denied errors. This does not prevent injection from occurring but prevents the attacker from leveraging it for privilege escalation, file reads, or cross-database enumeration.

***

## Web Application Firewall (WAF)

Inspects HTTP requests and blocks those containing known malicious patterns. A WAF is a supplementary control, not a primary defence, because determined attackers can bypass WAF rules through encoding, case variation, and comment insertion:

```
-- Blocked by WAF keyword rules:
UNION SELECT, INFORMATION_SCHEMA, LOAD_FILE, INTO OUTFILE, SLEEP()

-- Potential bypass attempts (may evade naive rules):
UnIoN SeLeCt
UN/**/ION SEL/**/ECT
UNION%20SELECT
```

ModSecurity (open-source) and Cloudflare (premium) are the most commonly deployed WAF solutions. Both maintain rulesets that block common SQLi patterns, but neither should be the only line of defence in a properly secured application.

***

## Defence Layer Summary

| Defence | Prevents Injection | Limits Impact | Notes |
|---------|------------------|--------------|-------|
| Parameterised queries | Yes (structural) | N/A | Primary fix, implement always |
| Input sanitisation | Partially | No | Secondary, DB-specific, bypassable |
| Input validation | Partially | No | Best when input format is strict |
| Least privilege | No | Yes | Limits what injected queries can do |
| WAF | Partially | No | Supplementary only, bypassable |
| Error suppression | No | Partially | Removes enumeration feedback |

The practical implementation priority is: parameterised queries everywhere first, least-privilege DB accounts always, input validation where input format is clearly defined, and WAF as an additional layer. Sanitisation functions alone without parameterised queries leave the application one encoding trick away from compromise.
