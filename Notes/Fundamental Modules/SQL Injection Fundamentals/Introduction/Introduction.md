# SQL Injection: Use Cases and Impact

SQL injection remains one of the most dangerous and widespread web vulnerabilities because it targets the database layer directly, not just the application. When it works, the impact goes far beyond reading a few records.

***

## How Injection Occurs

The vulnerability arises at the boundary between user input and SQL query construction. When a web application takes user-supplied data and concatenates it directly into a query string, the database receives a single string it cannot distinguish between intended structure and injected commands.

The entry point is almost always a quote character. A single quote `'` or double quote `"` closes the string context the developer opened, allowing the attacker to append arbitrary SQL:

```sql
-- Developer intended:
SELECT * FROM users WHERE username = 'INPUT' AND password = 'INPUT'

-- Attacker enters username: admin'--
SELECT * FROM users WHERE username = 'admin'-- AND password = ''
-- Everything after -- is a comment, password check is gone
```

Once inside the query structure, the attacker can reshape the logic entirely.

***

## Two Stages of Every SQLi Attack

Every successful SQL injection follows the same two-stage pattern:

1. Breaking out of the intended input context (using `'`, `"`, or other delimiters)
2. Injecting valid SQL that either modifies the original query or appends a new one

The second stage uses one of several techniques:

| Technique | How It Works |
|-----------|-------------|
| Comment injection (`--`, `#`) | Truncates the rest of the original query |
| Boolean logic (`OR 1=1`) | Makes a WHERE clause always true |
| UNION queries | Appends a second SELECT to retrieve data from other tables |
| Stacked queries (`;`) | Terminates the original and executes a new one |
| Subqueries | Embeds a query within the query to extract data |

***

## Impact by Severity

The real-world consequences of SQL injection span a wide range depending on database privileges and server configuration:

- **Credential theft**: Dump usernames and password hashes from the users table, which are then cracked or used in credential stuffing attacks
- **Authentication bypass**: Log in as any user, including administrators, without knowing their password
- **Privilege escalation**: Access admin panels or features restricted to specific user roles
- **Data manipulation**: Insert, update, or delete records, corrupting application data or planting backdoor accounts
- **File system access**: On misconfigured MySQL instances, `LOAD_FILE()` reads server files and `INTO OUTFILE` writes files to the web root, enabling webshell deployment
- **Full server compromise**: Writing a webshell to the server via `INTO OUTFILE` can hand the attacker remote code execution on the underlying OS

The escalation from SQLi to full server control, while not always possible, is a realistic path when database users are granted excessive privileges.

***

## Why Poor Privileges Make Everything Worse

The principle of least privilege is the most underutilised defence against SQLi impact. A web application only needs `SELECT`, `INSERT`, and `UPDATE` on specific tables. When the database connection runs as `root` or a superuser, a successful injection gains those same rights, including `DROP TABLE`, `FILE` operations, and the ability to create new database users.

Separating database users by function (read-only for public queries, write access only for specific operations, no file privileges for web-facing accounts) does not prevent the injection from occurring but dramatically limits what an attacker can do with it.

***

## The Prevention Hierarchy

Prevention operates at several layers, and all of them matter:

1. **Parameterised queries / prepared statements**: The structural fix. User input is passed as a typed parameter, never interpreted as SQL. This eliminates the injection vector entirely at the code level
2. **Input validation and sanitisation**: Reject or escape characters that have no business appearing in the expected input type
3. **Stored procedures**: Encapsulate query logic server-side, though these can still be vulnerable if they concatenate input internally
4. **Least-privilege database accounts**: Limits blast radius when injection does occur
5. **Web application firewall (WAF)**: Catches common payloads as a supplementary control, not a primary defence
6. **Error handling**: Never expose raw database error messages to users, as these reveal table names, column names, and query structure that directly assist an attacker

The rest of this module will build on these concepts, covering MySQL syntax, UNION-based extraction, blind injection, and reading/writing files, giving you a complete picture of how SQLi works from both the offensive and defensive perspective.
