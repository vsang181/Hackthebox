# Development Frameworks and APIs

Modern web applications are rarely built entirely from scratch. As applications grow in size and complexity, it becomes inefficient and error-prone to manually implement every feature and security control. This is where **development frameworks** come in. Frameworks provide structure, reusable components, and common patterns that allow developers to build applications faster and more consistently.

As you analyse real-world applications, you should assume that a framework is involved somewhere, even if it is not immediately obvious.

---

## Development Frameworks

Most web applications share a common set of requirements, such as:

* User registration and authentication
* Session handling
* Input validation and error handling
* Database access
* Routing and templating

Frameworks abstract these repetitive tasks so developers can focus on application logic instead of rebuilding the same components repeatedly.

Some commonly encountered frameworks include:

* **Laravel (PHP)**
  Often used in small to medium-sized applications. It provides strong conventions, built-in security features, and rapid development workflows.

* **Express (Node.js)**
  A lightweight framework commonly used for APIs and back-end services. You will frequently encounter it in JSON-based REST applications.

* **Django (Python)**
  A full-featured framework with strong defaults and built-in administrative tooling. It is widely used in large and mature applications.

* **Ruby on Rails (Ruby)**
  Built around convention over configuration. Many older but still widely deployed applications rely on Rails.

In practice, complex systems rarely rely on a single framework. It is common to see multiple services, each built using different frameworks, working together.

---

## APIs and Back-End Communication

Communication between the front end and the back end is usually handled through **APIs (Application Programming Interfaces)**. APIs define how different components of an application exchange data and trigger actions.

From a security perspective, APIs are critical because they often expose:

* Core business logic
* Sensitive actions
* Privileged functionality
* Endpoints not directly visible in the user interface

---

## Query Parameters

One of the simplest ways to pass data to the server is through HTTP request parameters.

Example using a GET request:

`/search.php?item=apples`

Example using a POST request:

```http
POST /search.php HTTP/1.1
Host: example.com

item=apples
```

Query parameters allow a single endpoint to behave differently depending on user input. However, they are also a common source of vulnerabilities if input is not validated or sanitised correctly.

---

## Web APIs

A **Web API** exposes back-end functionality so that other components (such as browsers or mobile applications) can interact with it remotely. APIs commonly return data in formats such as:

* JSON
* XML
* `application/x-www-form-urlencoded`
* Binary data

Two major API design approaches you will encounter are **SOAP** and **REST**.

---

## SOAP APIs

**SOAP (Simple Object Access Protocol)** uses XML to structure requests and responses. It follows strict formatting rules and is often used in enterprise environments where structured communication is required.

SOAP APIs are typically:

* Verbose
* Rigid in structure
* Less common in modern web applications

---

## REST APIs

**REST (Representational State Transfer)** is the most widely used API style today.

Typical REST characteristics include:

* Resource-based endpoints
* JSON-formatted responses
* Behaviour defined by HTTP methods

Example endpoint:

`/category/posts/`

Example response:

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

Common HTTP methods you should recognise immediately:

* **GET** – Retrieve data
* **POST** – Create new data (non-idempotent)
* **PUT** – Create or replace data (idempotent)
* **DELETE** – Remove data
