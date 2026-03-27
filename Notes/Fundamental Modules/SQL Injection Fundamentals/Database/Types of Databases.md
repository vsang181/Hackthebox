# Types of Databases

Databases split into two fundamental categories: relational and non-relational. The distinction matters for SQL injection because SQLi only applies to relational databases. Non-relational databases have their own injection class covered separately.

***

## Relational Databases

A relational database organises data into structured tables, each representing a single entity type. Tables are linked together through keys, allowing complex data relationships to be expressed efficiently without duplicating data.

The core concepts:

- **Schema**: The defined structure of the database, specifying what tables exist, what columns each has, and what data types each column holds
- **Primary key**: A column (usually `id`) that uniquely identifies each row in a table
- **Foreign key**: A column in one table that references the primary key of another, establishing the relationship between them
- **RDBMS**: The management system that enforces these relationships and handles queries across linked tables

A practical example of how keys link tables:

```
users table                    posts table
-----------                    -----------
id  | username | email         id | user_id | content
1   | jsmith   | j@mail.com    1  |    1    | "Hello world"
2   | adoe     | a@mail.com    2  |    1    | "Second post"
                               3  |    2    | "My first post"
```

The `user_id` in `posts` references `id` in `users`. To get all posts with their author's email, a single JOIN query retrieves data across both tables without duplicating email addresses in every post row. This normalisation is what makes relational databases both storage-efficient and fast for structured data.

Common relational databases include MySQL, PostgreSQL, Microsoft SQL Server, Oracle, and SQLite. MySQL is the focus of this module because it is the most common back-end for web applications.

***

## Non-Relational Databases (NoSQL)

Non-relational databases abandon the table/row/column model entirely in favour of flexible storage structures. They have no fixed schema, which makes them well suited for unstructured or rapidly changing data.

The four main NoSQL storage models:

| Model | Data Structure | Best Used For | Example DB |
|-------|---------------|--------------|-----------|
| Key-Value | Dictionary-style `{key: value}` pairs | Session data, caching | Redis, DynamoDB |
| Document-Based | JSON/BSON documents with nested fields | Content, user profiles | MongoDB, CouchDB |
| Wide-Column | Rows with dynamic columns per row | Time-series, analytics | Cassandra, HBase |
| Graph | Nodes and edges representing relationships | Social networks, recommendation engines | Neo4j |

A Key-Value store representing posts looks like this:

```json
{
  "100001": {
    "date": "01-01-2021",
    "content": "Welcome to this web application."
  },
  "100002": {
    "date": "02-01-2021",
    "content": "This is the first post on this web app."
  }
}
```

This is structurally identical to a Python dictionary or a PHP associative array. There are no columns, no joins, no schema enforcement. The application is responsible for knowing what fields a document might contain.

***

## Relational vs Non-Relational: Choosing Between Them

| Factor | Relational (SQL) | Non-Relational (NoSQL) |
|--------|-----------------|----------------------|
| Data structure | Well-defined, consistent | Flexible, variable |
| Scalability | Vertical (bigger server) | Horizontal (more servers) |
| Query language | Standardised SQL | Database-specific APIs |
| Relationships | Native (JOINs) | Application-managed |
| Consistency | Strong (ACID) | Often eventual |
| Best for | Structured business data | Large-scale, varied data |

Most web applications use relational databases for their core user and transaction data precisely because the structured schema and enforced relationships prevent inconsistency. NoSQL databases appear alongside them for specific use cases like caching (Redis), session storage, or content management where flexibility outweighs the need for strict structure.

***

## The SQLi Relevance

SQL injection targets the query language itself, so it only applies where SQL is used: relational databases. The attack exploits how `SELECT`, `WHERE`, `UNION`, and other SQL constructs work. A MongoDB backend using its own document query API has different injection characteristics entirely, covered under NoSQL injection in a separate module. For this module, MySQL is the target, and every technique covered builds on understanding how MySQL parses and executes SQL statements.
