# MySQL Fundamentals

MySQL is the most common database back-end in web applications, and understanding its syntax is a prerequisite for understanding how SQL injection payloads are constructed. Everything from authentication bypass to data extraction relies on being able to write valid MySQL that the database will execute.

***

## Connecting via Command Line

```bash
# Recommended: prompt for password (keeps it out of shell history)
mysql -u root -p

# Remote host with specific port
mysql -u root -h docker.hackthebox.eu -P 3306 -p
```

The flag case distinction matters: `-p` (lowercase) is for password, `-P` (uppercase) is for port. Passing the password directly as `-pPASSWORD` works but writes it to `.bash_history` and process listings, so always use the empty `-p` flag and enter it at the prompt instead.

***

## Database Operations

```sql
-- Create a new database
CREATE DATABASE users;

-- List all databases
SHOW DATABASES;

-- Switch to a database
USE users;

-- List tables in current database
SHOW TABLES;

-- View table structure
DESCRIBE logins;
```

SQL keywords are case-insensitive (`USE` and `use` are equivalent), but database and table names are case-sensitive on Linux systems. The convention of writing keywords in uppercase exists purely for readability.

***

## Creating Tables

A table definition specifies each column's name, data type, and constraints. The full `CREATE TABLE` statement from the module demonstrates every important property:

```sql
CREATE TABLE logins (
    id              INT          NOT NULL AUTO_INCREMENT,
    username        VARCHAR(100) UNIQUE NOT NULL,
    password        VARCHAR(100) NOT NULL,
    date_of_joining DATETIME     DEFAULT NOW(),
    PRIMARY KEY (id)
);
```

Each property serves a specific purpose:

| Property | Effect |
|---------|--------|
| `NOT NULL` | Rejects inserts that leave this column empty |
| `AUTO_INCREMENT` | Automatically assigns the next integer on each insert |
| `UNIQUE` | Rejects duplicate values in this column across all rows |
| `DEFAULT NOW()` | Populates the column with the current timestamp if no value is given |
| `PRIMARY KEY` | Marks this column as the unique row identifier, used for table relationships |

***

## Common MySQL Data Types

| Type | Used For | Example |
|------|---------|---------|
| `INT` | Whole numbers | `42`, `1000` |
| `VARCHAR(n)` | Variable-length strings up to n characters | `'admin'`, `'john.smith'` |
| `TEXT` | Long strings with no fixed length limit | Blog post content |
| `DATETIME` | Date and time combined | `2024-01-15 14:30:00` |
| `BOOLEAN` | True/false stored as 0 or 1 | `1` (active), `0` (inactive) |
| `FLOAT` / `DECIMAL` | Numbers with decimal points | `19.99` |

`VARCHAR(100)` is the most common type for credential fields. The length limit matters for SQLi: input exceeding 100 characters will be rejected by the database, which can affect payload construction if the injection point is a constrained field.

***

## The Information Schema: Critical for SQLi

MySQL maintains a special built-in database called `information_schema` that stores metadata about every other database on the server. It is visible in every `SHOW DATABASES` output and is the foundation of most SQL injection enumeration techniques:

```sql
-- List all databases on the server
SELECT schema_name FROM information_schema.schemata;

-- List all tables in a specific database
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'users';

-- List all columns in a specific table
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'logins';
```

During SQL injection, querying `information_schema` is how an attacker maps the entire database structure without any prior knowledge of table or column names. The existence of this schema is by design, as it allows legitimate applications to introspect the database, but it becomes a powerful reconnaissance tool once injection is achieved.

***

## Why Table Structure Matters for SQLi

The `DESCRIBE` output is directly relevant to crafting UNION-based injection payloads. UNION queries require the injected SELECT to return the same number of columns as the original query. Knowing a table's structure (four columns in the `logins` example) tells you how many columns to include in your UNION payload and what data types each position expects. This connection between basic MySQL knowledge and practical injection technique is what this section is building toward.
