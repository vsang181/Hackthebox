# Front End vs Back End

You will often hear terms like **front end**, **back end**, or **full-stack** when people talk about web applications. While they are closely related, they describe **very different parts of the same system**. Understanding where the front end ends and the back end begins is essential if you want to test web applications properly and reason about where vulnerabilities actually live 

Think of a web application as a conversation. The front end speaks *to you*. The back end decides *what is allowed to happen*.

---

## Front End (Client Side)

The **front end** is everything that runs inside your **web browser**. This is the part of the application you can see, interact with, inspect, and modify directly.

It is typically made up of:

* **HTML** – structure and content
* **CSS** – layout, styling, and visual behaviour
* **JavaScript** – logic, interaction, and dynamic behaviour

This code is downloaded to your browser and executed **in real time**. Because of that, you should always assume:

> Anything on the front end is fully under the attacker’s control.

---

### What the Front End Is Responsible For

From your perspective, the front end handles:

* Page layout and visual design
* User interaction (clicks, forms, navigation)
* Client-side validation and logic
* Animations and responsiveness

Modern front ends are expected to:

* Adapt to different screen sizes
* Work across browsers
* Support mobile devices
* Remain responsive under load

If a front end is poorly optimised, users often blame:

* The server
* The network
* Their internet connection

When in reality, the bottleneck is entirely **client-side**.

---

### Front End Development Is More Than Code

In addition to writing HTML, CSS, and JavaScript, front end work often includes:

* Visual concept and layout design
* **UI (User Interface)** design
* **UX (User Experience)** design

These decisions directly affect how users interact with the application and how easily security flaws can be abused.

![frontend-components](https://github.com/user-attachments/assets/f7c9a589-27e5-4322-b82b-ac3d3ef74232)

---

## Back End (Server Side)

The **back end** is where the real work happens. It runs on servers and executes code you normally **cannot see directly**.

Without a back end, a website is just a collection of static pages.

![backend-server](https://github.com/user-attachments/assets/d44c2a38-dfc0-4649-a6de-67219e60790c)

---

### Core Back End Components

Most web applications rely on four main back end components:

| Component                  | What It Does                                                                     |
| -------------------------- | -------------------------------------------------------------------------------- |
| **Back End Servers**       | Host the application and run the operating system (Linux, Windows, containers).  |
| **Web Servers**            | Handle HTTP(S) requests (Apache, NGINX, IIS).                                    |
| **Databases**              | Store and retrieve application data (MySQL, PostgreSQL, MSSQL, Oracle, MongoDB). |
| **Development Frameworks** | Implement application logic (Laravel, ASP.NET, Spring, Django, Express).         |

The back end:

* Processes requests
* Enforces authentication and authorisation
* Applies business logic
* Talks to databases and services
* Decides what data is returned to the user

This is where **most high-impact vulnerabilities live**.

---

## Deployment and Isolation

Back end components can be deployed in several ways:

* All components on **one server**
* Separate servers for application and database
* Containers (for example, Docker)
* Fully distributed environments

For example:

* The database may run in one container
* The web application in another
* Supporting services in additional containers

This separation helps reduce impact, but **only if access controls are implemented correctly**. Poor isolation gives a false sense of security.

---

## What the Back End Is Responsible For

Typical back end responsibilities include:

* Implementing core application logic
* Managing databases and data integrity
* Handling authentication and authorisation
* Exposing APIs to the front end
* Integrating third-party and cloud services
* Enforcing technical and business rules

If the back end trusts user input too much, no amount of front end polish will save the application.

---

## Securing Front End vs Back End

Even if you never see the back end source code, the application is **not safe by default**.

You may still be able to exploit it via:

* SQL Injection
* Command Injection
* File Inclusion
* Logic flaws
* Broken access control

For example:

* A poorly validated search function may allow SQL Injection
* A file upload feature may allow remote code execution
* A parameter may allow access to another user’s data

---

### Whitebox vs Blackbox Testing

Depending on access, you will test applications in different ways:

* **Whitebox Pentesting**
  You have source code access and perform detailed code review.

* **Blackbox Pentesting**
  You have no source code and must infer behaviour from responses.

Some applications are **open source**, meaning you may legally obtain the code. In other cases, vulnerabilities like **Local File Inclusion (LFI)** may allow you to retrieve back end source code, turning a blackbox test into something closer to whitebox.

---

## Common Developer Mistakes You Should Expect

As a penetration tester, you should assume mistakes exist. The following are **extremely common**:

1. Allowing invalid data into the database
2. Focusing only on the system as a whole
3. Inventing custom security mechanisms
4. Treating security as a final step
5. Storing passwords in plain text
6. Allowing weak passwords
7. Storing sensitive data unencrypted
8. Trusting client-side validation
9. Being overly optimistic about user behaviour
10. Accepting variables directly from URLs
11. Trusting third-party code blindly
12. Hard-coding backdoor accounts
13. Failing to properly handle SQL Injection
14. Allowing remote file inclusion
15. Insecure data handling
16. Improper encryption usage
17. Weak cryptographic systems
18. Ignoring human factors (layer 8)
19. Failing to review user actions
20. Misconfigured Web Application Firewalls

These mistakes are not rare. They are routine.

---

## How This Maps to OWASP Top 10

Many of the mistakes above directly lead to the **OWASP Top 10** vulnerabilities:

1. Broken Access Control
2. Cryptographic Failures
3. Injection
4. Insecure Design
5. Security Misconfiguration
6. Vulnerable and Outdated Components
7. Identification and Authentication Failures
8. Software and Data Integrity Failures
9. Security Logging and Monitoring Failures
10. Server-Side Request Forgery (SSRF)

You will encounter these repeatedly, across different technologies and industries.
