# Core SQL Statements

These five statements (INSERT, SELECT, DROP, ALTER, UPDATE) form the foundation of SQL interaction. For SQL injection purposes, SELECT is the most critical for data extraction, while INSERT, UPDATE, and DROP become relevant when write permissions are available.

***

## INSERT

Adds new records to a table. Three syntax variations:

```sql
-- All columns in order (must match every column)
INSERT INTO logins VALUES(1, 'admin', 'p@ssw0rd', '2020-07-02');

-- Specific columns only (skips AUTO_INCREMENT and DEFAULT columns)
INSERT INTO logins(username, password) VALUES('administrator', 'adm1n_p@ss');

-- Multiple records in one statement
INSERT INTO logins(username, password) VALUES ('john', 'john123!'), ('tom', 'tom123!');
```

Skipping a `NOT NULL` column without a `DEFAULT` value will throw an error. This is relevant to SQLi because stacked injection queries that attempt INSERT will fail if the payload does not satisfy all column constraints.

***

## SELECT

The most important statement for SQL injection. Retrieves data from one or more tables:

```sql
-- All columns, all rows
SELECT * FROM logins;

-- Specific columns only
SELECT username, password FROM logins;

-- Column from a specific database (fully qualified)
SELECT users.username FROM users.logins;
```

The `*` wildcard is what makes UNION-based injection powerful: if the original query uses `SELECT *`, your injected UNION only needs to match the column count, not specific names. Understanding SELECT also explains why `information_schema` queries during injection look like:

```sql
SELECT table_name FROM information_schema.tables WHERE table_schema = 'users';
```

This is just a standard SELECT against MySQL's built-in metadata tables.

***

## DROP

Permanently and irreversibly deletes tables or databases:

```sql
DROP TABLE logins;
DROP DATABASE users;
```

There is no confirmation prompt and no undo. This is why the classic `'; DROP TABLE users;--` injection is both dangerous and well-known. In a penetration test, executing DROP statements against a production database is strictly out of scope unless explicitly authorised, as it causes data loss that cannot be reversed without backups.

***

## ALTER

Modifies the structure of an existing table without touching its data:

```sql
-- Add a new column
ALTER TABLE logins ADD newColumn INT;

-- Rename a column
ALTER TABLE logins RENAME COLUMN newColumn TO newerColumn;

-- Change a column's data type
ALTER TABLE logins MODIFY newerColumn DATE;

-- Remove a column entirely
ALTER TABLE logins DROP newerColumn;
```

ALTER requires elevated database privileges. A web application's service account typically has no ALTER permission, so this statement rarely appears in SQLi attack chains unless the compromised account has DBA-level rights.

***

## UPDATE

Modifies existing records based on a condition:

```sql
-- Update all passwords where id > 1
UPDATE logins SET password = 'change_password' WHERE id > 1;

-- Update a single record by username
UPDATE logins SET password = 'newpass' WHERE username = 'admin';
```

The `WHERE` clause is critical. Without it, every row in the table is updated:

```sql
-- This changes ALL passwords in the table
UPDATE logins SET password = 'hacked';
```

In a SQL injection context with write access, UPDATE is the most surgically useful write statement: an attacker can reset an admin password to a known value, grant themselves admin status, or unlock a locked account, all without the destructive irreversibility of DROP.

***

## Statement Summary and SQLi Relevance

| Statement | Primary Use | SQLi Relevance |
|-----------|------------|----------------|
| `SELECT` | Read data | Core of all data extraction attacks |
| `INSERT` | Add records | Create backdoor admin accounts |
| `UPDATE` | Modify records | Change passwords, escalate privileges |
| `DROP` | Delete tables/DBs | Destructive, rarely used in assessments |
| `ALTER` | Change table structure | Requires DBA privileges, uncommon in SQLi |

The `WHERE` clause appears in SELECT, UPDATE, and DELETE statements and is the injection point in the majority of real-world SQLi vulnerabilities. The next section builds directly on this, covering how WHERE conditions are manipulated through boolean logic and comparison operators to both filter data and form the basis of authentication bypass attacks.
