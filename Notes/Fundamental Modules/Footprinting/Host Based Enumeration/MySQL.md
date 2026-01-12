# MySQL

MySQL is an open-source **relational database management system (RDBMS)** maintained by Oracle. It stores data in **tables** (rows and columns) and is controlled using **SQL**. It follows a **client-server** model:

* **MySQL server:** stores and manages data, processes queries.
* **MySQL client(s):** connect to the server to run SQL queries.

Databases are commonly backed up or exported into `.sql` files (for example `wordpress.sql`).

---

## Where MySQL is commonly used

MySQL is heavily used for **dynamic web applications** (for example WordPress). In common stacks:

* **LAMP:** Linux, Apache, MySQL, PHP
* **LEMP:** Linux, Nginx, MySQL, PHP

Applications may store:

* Users, roles, and permissions
* Email addresses and profile data
* Password hashes (ideally hashed by the application layer, not stored as plain text)
* Content such as posts, forms, and metadata

MariaDB is a well-known fork of MySQL and is largely compatible in day-to-day usage.

---

## Default Configuration

MySQL typically listens on **TCP/3306**.

A common config location on Debian/Ubuntu is:

* `/etc/mysql/mysql.conf.d/mysqld.cnf`

Key defaults usually include:

* `port = 3306`
* `socket = /var/run/mysqld/mysqld.sock`
* `datadir = /var/lib/mysql`
* The daemon running as a dedicated user (often `mysql`)

---

## Dangerous Settings and Why They Matter

Security-relevant configuration issues often fall into three buckets:

### 1) Credentials and access exposure

* `user` / `password` / admin or bind address settings may be stored in plain text in config files.
* If file permissions are weak, **anyone who can read configs** (via LFI, backup exposure, low-priv shell) may obtain DB credentials.

### 2) Verbose error and debug output

* `debug`, `sql_warnings`, or overly verbose logging can leak details that are useful for attackers:

  * Schema names and table names
  * Query structure
  * Application behaviour and input handling
* In web apps, these leaks often show up during failed requests and can assist SQL injection attempts.

### 3) Import/export controls

* `secure_file_priv` controls where MySQL can read/write files for import/export.
* Overly permissive settings can turn data export or file-write features into a bigger risk if an attacker obtains DB access.

---

# Footprinting MySQL

## 1) Scan for the service

MySQL usually runs on **3306**, so a service/version scan commonly looks like:

* `nmap -sV -sC -p3306 --script mysql* <target>`

Important note: automated NSE scripts can produce **false positives** (for example “root with empty password”), so always verify manually.

---

## 2) Manually confirm access

If you attempt to connect without a password and it is rejected, that confirms authentication is enforced:

```bash
mysql -u root -h <IP>
```

If you have credentials (guessed, found, or leaked), you can authenticate and enumerate:

```bash
mysql -u root -p<password> -h <IP>
```

(There must be no space between `-p` and the password.)

---

## 3) Useful MySQL enumeration commands (inside the client)

Connection and basic checks:

* `select version();`
* `show databases;`

Move through data:

* `use <database>;`
* `show tables;`
* `show columns from <table>;`
* `select * from <table>;`
* `select * from <table> where <column> = "<string>";`

---

## System databases you will commonly see

* `information_schema`
  Metadata views (standardised style; typically derived from internal structures).

* `mysql`
  Core system tables including users/privileges.

* `performance_schema`
  Performance and instrumentation data.

* `sys`
  Convenience views built on top of performance_schema to make analysis easier.

These are useful for identifying how the DB is being used, what hosts connect, and sometimes what user accounts exist, depending on permissions.
