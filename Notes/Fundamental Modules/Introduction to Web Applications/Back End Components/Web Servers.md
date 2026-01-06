# Web Servers

A **web server** is a back-end application that sits on a server and is responsible for handling all **HTTP and HTTPS traffic** coming from client-side browsers. Its job is to receive requests, route them to the correct resources or application logic, and then send the appropriate response back to the client 

Most web servers listen on:

* **TCP port 80** (HTTP)
* **TCP port 443** (HTTPS)

From your perspective as a tester, the web server is often the **first point of contact** with the back end and a critical component in how requests are processed, filtered, or rejected.

![web-server-requests](https://github.com/user-attachments/assets/a3734ea9-1b09-4fe9-b0ab-0a2141d32e9c)

---

## How a Web Server Works (High-Level Flow)

A typical request flow looks like this:

1. You send an HTTP request from a browser or tool
2. The web server receives the request
3. It decides how to handle it:

   * Serve a static file
   * Pass it to application logic
   * Reject it based on rules or configuration
4. It returns an HTTP response with:

   * A status code
   * Headers
   * Optional content (HTML, JSON, files, etc.)

The response tells the client **what happened** and **how to interpret the result**.

---

## Common HTTP Response Codes You Will See

Web servers use HTTP status codes to communicate outcomes. You should learn to recognise these instinctively.

### Successful Responses

* **200 OK** – The request was successful

### Redirection Responses

* **301 Moved Permanently** – Resource has a new permanent location
* **302 Found** – Resource has a temporary new location

### Client Error Responses

* **400 Bad Request** – Invalid request syntax
* **401 Unauthorized** – Authentication required or failed
* **403 Forbidden** – Access is denied
* **404 Not Found** – Resource does not exist
* **405 Method Not Allowed** – HTTP method is not permitted
* **408 Request Timeout** – Client took too long to send a request

### Server Error Responses

* **500 Internal Server Error** – Generic server-side failure
* **502 Bad Gateway** – Invalid response from upstream server
* **504 Gateway Timeout** – Upstream server did not respond in time

These codes are extremely valuable during testing because they often reveal:

* Access control behaviour
* Application routing logic
* Misconfigurations
* Error handling weaknesses

---

## User Input and Request Handling

Web servers accept many types of input within HTTP requests, including:

* Form data
* URL parameters
* JSON bodies
* File uploads (binary data)

Once a request is received, the web server:

* Parses the request
* Applies routing rules
* Forwards it to the correct application component
* Returns the final response

The files and scripts it routes requests to form the **core of the web application**.

---

## Interacting with Web Servers from the Command Line

You do not need a browser to talk to a web server.

For example, using `curl` to request only headers:

```bash
curl -I https://academy.hackthebox.com
```

This shows:

* HTTP version
* Status code
* Response headers

Requesting the full page content:

```bash
curl https://academy.hackthebox.com
```

This returns the raw HTML sent by the server. Tools like this are essential for:

* Automation
* Enumeration
* Debugging
* Bypassing client-side restrictions

---

## Popular Web Servers You Will Encounter

Although it is possible to write a custom web server in languages like Python or JavaScript, most applications rely on **battle-tested, optimised web servers**.

### Apache (httpd)

Apache is the **most widely used web server** on the internet.

<img width="1200" height="458" alt="apache" src="https://github.com/user-attachments/assets/cd1d10b0-73d4-422b-9986-9e771fc7295e" />

Key points:

* Open source
* Commonly pre-installed on Linux
* Available on Windows and macOS
* Highly modular via extensions

Apache is often paired with:

* PHP (via `mod_php`)
* Python
* Perl
* .NET
* CGI-based scripts

It is popular because:

* It is well documented
* It is flexible
* It is easy to configure

You will frequently see Apache in:

* Startups
* Small to medium organisations
* Legacy environments

---

### NGINX

NGINX is the **second most common web server** and dominates high-traffic environments.

<img width="1921" height="645" alt="nginx" src="https://github.com/user-attachments/assets/168c0e02-d163-4975-a4d8-d1f4488add58" />

Key characteristics:

* Event-driven, asynchronous architecture
* Very low memory and CPU usage
* Excellent performance under heavy load

Because of this, NGINX is widely used by:

* Large-scale platforms
* Content-heavy websites
* High-concurrency applications

NGINX is commonly deployed as:

* A primary web server
* A reverse proxy
* A load balancer
* A TLS termination point

As a tester, you will often encounter NGINX in modern production environments.

---

### IIS (Internet Information Services)

IIS is Microsoft’s web server and primarily runs on **Windows Server**.

<img width="396" height="120" alt="iis" src="https://github.com/user-attachments/assets/aea1a6d3-0027-48e7-8636-35284bc7d7ab" />

Key features:

* Tight integration with Windows and Active Directory
* Strong support for .NET applications
* Built-in Windows authentication mechanisms

IIS is commonly found in:

* Enterprise environments
* Organisations dependent on Active Directory
* Microsoft-centric technology stacks

Understanding IIS is essential if you test:

* Corporate applications
* Internal portals
* Enterprise web services

---

## Other Web Servers and Runtimes

Beyond the big three, you will also encounter:

* **Apache Tomcat** – Java-based web applications
* **Node.js** – JavaScript-based back ends
* Embedded servers inside frameworks

Each server type has its own:

* Configuration style
* Default behaviours
* Common misconfigurations

Recognising which server you are dealing with helps you:

* Predict file locations
* Identify default credentials or paths
* Focus your enumeration efficiently

---

## Why Web Servers Matter to You

The web server is not just a delivery mechanism. It often:

* Enforces access controls
* Applies security headers
* Handles routing and redirects
* Exposes error messages
* Integrates with authentication systems

Misconfigurations at this level can lead to:

* Directory traversal
* Information disclosure
* Authentication bypass
* File upload vulnerabilities
* Logic flaws

If you understand how web servers behave, you will start to see patterns quickly—and many vulnerabilities become much easier to reason about and exploit.
