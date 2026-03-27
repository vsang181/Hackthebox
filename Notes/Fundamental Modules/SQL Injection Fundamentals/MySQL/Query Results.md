# Query Results Control

Controlling query output is fundamental to both legitimate database use and SQL injection. The clauses covered here: ORDER BY, LIMIT, WHERE, and LIKE, each appear in real-world injection scenarios and need to be understood both for constructing payloads and for interpreting what the database returns.

***

## ORDER BY

Sorts results by one or more columns. Default direction is ascending:

```sql
-- Ascending (default)
SELECT * FROM logins ORDER BY password;

-- Descending
SELECT * FROM logins ORDER BY password DESC;

-- Multiple columns: primary sort by password DESC, secondary by id ASC
SELECT * FROM logins ORDER BY password DESC, id ASC;
```

ORDER BY is directly relevant to SQL injection in one critical way: during UNION-based injection, you can use `ORDER BY` to enumerate the number of columns in the original query without knowing the table structure. By incrementing the column number until the query errors out, you identify the exact column count needed for a valid UNION:

```sql
-- Testing column count via ORDER BY
' ORDER BY 1--    -- succeeds
' ORDER BY 2--    -- succeeds
' ORDER BY 3--    -- succeeds
' ORDER BY 4--    -- ERROR: column 4 doesn't exist
-- Therefore the original query returns 3 columns
```

***

## LIMIT and OFFSET

Controls how many rows are returned and where in the result set to start:

```sql
-- Return only the first 2 rows
SELECT * FROM logins LIMIT 2;

-- Skip 1 row, then return 2 rows (offset, count)
SELECT * FROM logins LIMIT 1, 2;
```

The offset is zero-indexed: `LIMIT 1, 2` skips the first row (index 0) and returns rows at index 1 and 2. This is important for blind SQL injection, where you extract data one row at a time by incrementing the offset:

```sql
-- Extract usernames one row at a time
SELECT username FROM users LIMIT 0,1  -- first user
SELECT username FROM users LIMIT 1,1  -- second user
SELECT username FROM users LIMIT 2,1  -- third user
```

***

## WHERE Clause

Filters rows based on conditions. The engine evaluates each row against the condition and returns only those where it evaluates to true:

```sql
-- Numeric comparison
SELECT * FROM logins WHERE id > 1;

-- Exact string match (case-insensitive in MySQL by default)
SELECT * FROM logins WHERE username = 'admin';
```

WHERE operators available in MySQL:

| Operator | Meaning | Example |
|---------|---------|---------|
| `=` | Equal | `WHERE id = 1` |
| `!=` or `<>` | Not equal | `WHERE id != 1` |
| `>` / `<` | Greater / less than | `WHERE id > 2` |
| `>=` / `<=` | Greater or equal / less or equal | `WHERE id >= 3` |
| `AND` | Both conditions true | `WHERE id > 1 AND id < 4` |
| `OR` | Either condition true | `WHERE id = 1 OR id = 3` |
| `NOT` | Negates condition | `WHERE NOT id = 1` |
| `BETWEEN` | Within a range | `WHERE id BETWEEN 2 AND 4` |
| `IN` | Matches any in a list | `WHERE id IN (1, 3)` |
| `IS NULL` | Column has no value | `WHERE password IS NULL` |

The `OR` operator is the foundation of the most common authentication bypass payload. When injected into a login query:

```sql
-- Normal query
SELECT * FROM users WHERE username = 'admin' AND password = 'wrong';
-- Returns 0 rows, login fails

-- Injected: input is: ' OR '1'='1
SELECT * FROM users WHERE username = '' OR '1'='1' AND password = '';
-- '1'='1' is always true, returns all rows, login succeeds
```

***

## LIKE Clause

Pattern matching for string comparisons using two wildcard characters:

```sql
-- % matches zero or more characters
SELECT * FROM logins WHERE username LIKE 'admin%';
-- Matches: admin, administrator, admin123, adminX

-- _ matches exactly one character
SELECT * FROM logins WHERE username LIKE '___';
-- Matches only usernames with exactly 3 characters: tom, jay, etc.

-- Combining both
SELECT * FROM logins WHERE username LIKE '_a%';
-- Matches any username where second character is 'a': jane, gary, ...
```

LIKE is useful in SQL injection for two scenarios. First, when probing for sensitive data where you only know part of a value. Second, in boolean-based blind injection, LIKE can be used to extract data character by character:

```sql
-- Is the first character of the admin password 'p'?
SELECT * FROM users WHERE username='admin' AND password LIKE 'p%';
-- If rows returned: first char is 'p'
-- If no rows: first char is not 'p', try next character
```

***

## Clause Execution Order

Understanding the logical order MySQL processes these clauses helps predict query behaviour:

```
FROM → WHERE → SELECT → ORDER BY → LIMIT
```

WHERE filters happen before SELECT retrieves columns, which is why you can filter on columns not included in the SELECT list. LIMIT applies last, after sorting, which is why `ORDER BY id DESC LIMIT 1` reliably returns the highest ID row. This ordering matters when constructing subqueries and nested injection payloads where the sequence of operations determines what data surfaces.
