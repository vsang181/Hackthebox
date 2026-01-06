# Common Web Vulnerabilities

When you assess a web application—especially an **internally developed system** or a public application with no known exploits—you will often need to identify vulnerabilities **manually**. Many real-world issues are not caused by the application itself, but by **developer misconfigurations**, unsafe assumptions, or insecure implementation choices.

The vulnerability categories below are among the **most frequently encountered in web applications** and form a significant portion of the **OWASP Top 10**. You will see these repeatedly throughout your learning and in real assessments.

---

## Broken Authentication and Broken Access Control

Broken Authentication and Broken Access Control are some of the **most dangerous vulnerabilities** you can encounter in a web application.

### Broken Authentication

Broken Authentication occurs when flaws in the authentication mechanism allow you to:

* Log in without valid credentials
* Bypass login checks entirely
* Escalate privileges (for example, a normal user becoming an administrator)

This often happens due to:

* Poor input validation
* Weak session handling
* Unsafe authentication logic

A classic example is an authentication bypass where user input is directly embedded into a database query. In some applications, supplying a crafted value such as a logical condition in a login field can cause the authentication check to always succeed.

---

### Broken Access Control

Broken Access Control occurs when you can access **resources or functionality you are not authorised to use**.

Examples include:

* A normal user accessing an admin panel
* Accessing restricted pages by guessing URLs
* Modifying object identifiers to access another user’s data

Access control issues often exist even when authentication works correctly. This is why you should always test **authorisation separately from authentication**.

---

## Malicious File Upload

File upload functionality is a very common attack surface.

If an application allows file uploads but does not correctly validate:

* File type
* File extension
* File contents
* Upload location

You may be able to upload a **malicious script** (for example, a PHP web shell) and execute commands on the back-end server.

Even when developers attempt to implement validation, it is often:

* Incomplete
* Performed only on the client side
* Based on file extensions alone

These checks are frequently bypassable. One common technique is using **double extensions** (for example, `shell.php.jpg`) to trick the application into accepting executable files.

---

## Command Injection

Many web applications execute **operating system commands** as part of their functionality. Examples include:

* Installing plugins
* Running system utilities
* Performing network checks

If user input is passed into these commands without proper sanitisation, you may be able to **inject additional commands**.

This allows you to:

* Execute arbitrary commands on the server
* Read sensitive files
* Pivot further into the system

Command Injection is particularly dangerous because it often leads directly to **remote code execution**.

This vulnerability is widespread because:

* Input validation is difficult to implement correctly
* Developers often rely on weak filtering logic
* Blacklists are easy to bypass

---

## SQL Injection (SQLi)

SQL Injection is one of the **most well-known and impactful web vulnerabilities**.

It occurs when user input is embedded directly into a SQL query without proper handling. For example:

```php
$query = "SELECT * FROM users WHERE name LIKE '%$searchInput%'";
```

If the input is not validated or parameterised, you may be able to:

* Modify the logic of the query
* Bypass authentication
* Extract sensitive data
* Modify or delete database contents

In severe cases, SQL Injection can lead to:

* Full database compromise
* Credential exposure
* Remote command execution through database features

SQL Injection vulnerabilities are especially dangerous because databases often contain **the most valuable data** in an application.
