# Databases

When you work with web applications, you will almost always interact with a back-end database. Databases are used to store and manage application data such as images, uploaded files, posts, comments, configuration data, and user information like usernames and passwords. This is what allows a web application to serve dynamic content that changes depending on who the user is and what they are doing.

You will notice that databases are chosen based on several practical factors. As you study different applications, keep asking yourself why a specific database was selected. Common decision points include performance when reading and writing data, how well the database scales as the application grows, how efficiently it stores large datasets, and the overall cost of maintenance and licensing. :contentReference[oaicite:0]{index=0}

---

## Relational Databases (SQL)

Relational databases, often referred to as SQL databases, organise data into **tables** made up of **rows** and **columns**. Each table usually contains one or more **keys**, which allow tables to be linked together.

A simple example you will encounter frequently is a `users` table with columns such as `id`, `username`, `first_name`, and `last_name`. The `id` column acts as a primary key. Another table, such as `posts`, might include `id`, `user_id`, `date`, and `content`.

By linking `users.id` to `posts.user_id`, you can retrieve user information for each post without duplicating user details in every record. This structure is known as a **database schema**.

Relational databases are particularly effective when the data is well structured and relationships between different entities are clearly defined. This makes them reliable, fast, and easy to query, even when dealing with large datasets.

### Common SQL Databases

| Database | Notes |
|--------|------|
| MySQL | Very widely used, open-source, and free. Common in Linux-based stacks. |
| MSSQL | Microsoft’s SQL database, commonly paired with Windows Server and IIS. |
| Oracle | Enterprise-focused, highly reliable, but often expensive. |
| PostgreSQL | Open-source and extensible, with advanced features and strong standards compliance. |

You may also come across SQLite, MariaDB, Amazon Aurora, and Azure SQL in real-world environments.

---

## Non-Relational Databases (NoSQL)

Non-relational, or NoSQL, databases take a very different approach. They do not rely on tables, fixed schemas, or predefined relationships. Instead, they store data using flexible models that are easier to scale and adapt when the structure of the data is not fully known in advance.

This flexibility makes NoSQL databases a strong choice when dealing with unstructured or rapidly changing data.

### Common NoSQL Storage Models

- **Key–Value**
- **Document-Based**
- **Wide-Column**
- **Graph**

In a **Key–Value** model, data is stored as pairs where a unique key maps directly to a value. The value is often represented using JSON or XML.

Example (JSON):

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
````

If you have worked with dictionaries or maps in languages like Python or PHP, this structure should feel familiar.

### Common NoSQL Databases

| Database         | Notes                                                 |
| ---------------- | ----------------------------------------------------- |
| MongoDB          | Document-based, open-source, and extremely popular.   |
| ElasticSearch    | Optimised for searching and analysing large datasets. |
| Apache Cassandra | Designed for high scalability and fault tolerance.    |

Other NoSQL technologies you may encounter include Redis, Neo4j, CouchDB, and Amazon DynamoDB.

---

## Databases in Web Applications

Modern programming languages and frameworks make database integration relatively straightforward. Once a database server is installed and configured on the back-end system, applications can connect to it and begin storing and retrieving data.

For example, in a PHP application using MySQL, you may see a connection established like this:

```php
$conn = new mysqli("localhost", "user", "pass");
```

Creating a database might look like this:

```php
$sql = "CREATE DATABASE database1";
$conn->query($sql);
```

Once connected, the application can query data directly from the database:

```php
$searchInput = $_POST['findUser'];
$query = "SELECT * FROM users WHERE name LIKE '%$searchInput%'";
$result = $conn->query($query);
```

At this stage, you should pause and think like a penetration tester. Although this code works, directly inserting user input into database queries can introduce serious vulnerabilities, such as **SQL Injection**. As you continue your studies, always pay close attention to how user input is handled and whether secure coding practices, such as parameterised queries, are being used.

Databases are powerful tools, but when misused, they often become one of the most critical attack surfaces in a web application.
