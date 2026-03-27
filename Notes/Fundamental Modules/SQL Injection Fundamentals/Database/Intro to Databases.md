# Introduction to Databases

A database is an organised collection of data that a web application reads from and writes to in real time. Every dynamic web application, from a simple login form to a complex e-commerce platform, relies on a database to persist and retrieve information. Understanding the database layer is foundational to understanding SQL injection, because the attack targets the query mechanism directly.

***

## From File-Based Storage to DBMS

Early applications stored data in flat files on the server's filesystem. As data volumes grew, this became impractical: file-based storage has no concurrency management, no structured query capability, and degrades severely in performance at scale. This drove the adoption of dedicated Database Management Systems (DBMS), which abstract the storage layer and provide structured, reliable access to data.

Modern DBMS types each suit different data models:

| Type | Description | Examples |
|------|-------------|---------|
| Relational (RDBMS) | Data in structured tables with defined relationships | MySQL, PostgreSQL, MSSQL |
| NoSQL | Flexible document or key-value storage, no fixed schema | MongoDB, Redis, Cassandra |
| Graph-based | Stores relationships between entities as nodes and edges | Neo4j, ArangoDB |
| Key/Value stores | Simple lookup by unique key, extremely fast | Redis, DynamoDB |

This module focuses on MySQL, which is an RDBMS and the most common database type behind web applications.

***

## Core DBMS Features

Five properties make a DBMS suitable for production web applications:

- **Concurrency**: Multiple users can read and write simultaneously without data corruption. The DBMS manages locks and transaction isolation to prevent race conditions
- **Consistency**: All data obeys defined rules and constraints at all times. A transaction either completes fully or rolls back entirely, never leaving data in a partial state
- **Security**: Fine-grained access control through user accounts and permissions. A web application user account can be restricted to specific databases, tables, and even operation types
- **Reliability**: Point-in-time backups and transaction logs allow recovery from data loss or corruption
- **SQL interface**: A standardised, human-readable query language that abstracts the underlying storage implementation

***

## Two-Tier Architecture

Most web applications use a two-tier architecture that separates the user-facing layer from the data layer:

```
[Browser / Client]
       |
       | HTTP requests
       v
[Tier 1: Web Application]
  - Handles user interaction
  - Processes form data
  - Builds SQL queries
       |
       | SQL queries via DB driver/API
       v
[Tier 2: DBMS]
  - Executes queries
  - Returns result sets or errors
  - Enforces access controls
```

Tier 1 (the web application) receives user input, constructs SQL queries using that input, and sends them to Tier 2 (the DBMS). The DBMS executes the query and returns results. The web application then formats those results into the HTTP response the user sees.

The security implication is direct: the DBMS trusts whatever the application sends it. If the application forwards unsanitised user input as part of a query, the DBMS has no way to know that the query has been manipulated. It simply executes what it receives.

***

## Why This Architecture Creates the SQLi Risk

The two-tier model places the responsibility for query safety entirely on the application layer. The DBMS itself has no awareness of what the user originally typed versus what the developer intended to send. This is why SQL injection is an application-layer vulnerability, not a database vulnerability: the database behaves correctly, it executes exactly what it is given. The flaw is in how Tier 1 constructs the query before passing it to Tier 2.

Understanding this architecture also explains why the fix lives at the application layer too. Parameterised queries instruct the DBMS driver to send the query structure and the user input separately, so the database can distinguish between code and data before execution. No amount of database-side configuration alone can prevent injection if the application is concatenating user input into query strings.

***

## Structured Query Language (SQL)

SQL is the standardised language used to interact with relational databases. It is divided into sub-languages by function:

| Sub-language | Purpose | Example Statements |
|-------------|---------|-------------------|
| DDL (Data Definition) | Define structure | `CREATE`, `ALTER`, `DROP` |
| DML (Data Manipulation) | Read and write data | `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| DCL (Data Control) | Manage permissions | `GRANT`, `REVOKE` |
| TCL (Transaction Control) | Manage transactions | `COMMIT`, `ROLLBACK` |

For SQL injection, DML statements are the primary focus, particularly `SELECT` for data extraction and `INSERT`/`UPDATE` for data manipulation. Understanding how `SELECT` constructs result sets, how `WHERE` clauses filter rows, and how `UNION` combines results from multiple queries is the direct foundation for the injection techniques covered in this module.
