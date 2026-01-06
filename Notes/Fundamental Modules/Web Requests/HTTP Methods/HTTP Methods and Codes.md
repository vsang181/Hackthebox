# HTTP Methods and Codes

HTTP supports multiple methods for accessing a resource. These methods allow a client (such as a browser or `cURL`) to send information, forms, or files to a server and instruct the server how to process the request and respond.

As seen in previous sections:

* When using **cURL with `-v`**, the HTTP method appears in the first request line (e.g. `GET / HTTP/1.1`)
* In **browser DevTools**, the HTTP method is displayed in the **Method** column
* The server response includes an **HTTP status code** indicating the result of the request

---

## Request Methods

The following are some of the most commonly used HTTP methods:

| Method  | Description                                                                                                       |
| ------- | ----------------------------------------------------------------------------------------------------------------- |
| GET     | Requests a specific resource. Additional data can be passed using query strings in the URL (e.g. `?param=value`). |
| POST    | Sends data to the server in the request body. Commonly used for forms, authentication, and file uploads.          |
| HEAD    | Requests only the response headers that would be returned by a GET request. No response body is included.         |
| PUT     | Creates or replaces a resource on the server. If improperly secured, it can allow malicious file uploads.         |
| DELETE  | Deletes a specified resource on the server. Misconfiguration may result in Denial of Service (DoS).               |
| OPTIONS | Returns information about the server, including supported HTTP methods.                                           |
| PATCH   | Applies partial modifications to an existing resource.                                                            |

> **Note:**
> Most modern web applications primarily use **GET** and **POST**.
> Applications exposing **REST APIs** commonly rely on **PUT**, **PATCH**, and **DELETE** for data manipulation.

The availability of these methods depends on server and application configuration.
A complete list of HTTP methods can be found in the official HTTP specification.

---

## Status Codes

HTTP status codes indicate the outcome of a client’s request. They are grouped into five major classes:

| Class | Description                                                   |
| ----- | ------------------------------------------------------------- |
| 1xx   | Informational responses that do not affect request processing |
| 2xx   | Successful request processing                                 |
| 3xx   | Redirection responses                                         |
| 4xx   | Client-side errors (invalid requests or unauthorized access)  |
| 5xx   | Server-side errors                                            |

---

## Common HTTP Status Codes

| Code                      | Description                                                             |
| ------------------------- | ----------------------------------------------------------------------- |
| 200 OK                    | Request succeeded and the response body contains the requested resource |
| 302 Found                 | Redirects the client to another URL                                     |
| 400 Bad Request           | Malformed or invalid request                                            |
| 403 Forbidden             | Client does not have permission to access the resource                  |
| 404 Not Found             | Requested resource does not exist                                       |
| 500 Internal Server Error | Server encountered an unexpected condition                              |

> **Note:**
> In addition to standard HTTP status codes, some platforms (such as Cloudflare or AWS) introduce proprietary status codes for platform-specific behavior.

A full list of standard HTTP status codes can be found in the official HTTP documentation.
