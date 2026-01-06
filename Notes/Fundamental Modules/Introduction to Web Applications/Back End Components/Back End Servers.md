# Back End Servers

When you hear the term **back end server**, think of the **actual machine (or virtual machine)** that does the real work behind a web application. This is where the operating system runs, processes execute, services listen on ports, and data is stored or retrieved. Unlike the front end, this is not abstract—it is a real system with real files, users, permissions, and network exposure.

From an architectural point of view, the back end server typically sits in the **data access and application layers**, even though it physically hosts much more than just data.

![backend-server(1)](https://github.com/user-attachments/assets/6e53b10c-bad1-49bd-b12c-094eb6247379)

---

## What a Back End Server Really Is

At its simplest, a back end server consists of:

* **Hardware or virtualised hardware**
* An **operating system** (usually Linux or Windows)
* A collection of services that together power the application

This server is responsible for:

* Running application code
* Handling incoming requests
* Talking to databases
* Enforcing authentication and authorisation
* Performing background tasks and scheduled jobs

If something breaks at this level, the entire application usually suffers.

---

## Core Software Components on a Back End Server

Most back end servers host three critical software components:

### 1. Web Server

This is the component that:

* Listens for HTTP or HTTPS requests
* Serves static content
* Forwards dynamic requests to the application logic

Common examples you will encounter:

* Apache
* NGINX
* IIS

---

### 2. Application / Development Framework

This is where the **business logic** lives.

It handles:

* Request processing
* Input validation
* Authentication and authorisation
* Communication with databases and services

Typical technologies include:

* PHP
* Java
* C#
* Python
* Node.js

This layer is where many **logic flaws and high-impact vulnerabilities** are introduced.

---

### 3. Database

The database stores application data such as:

* User accounts
* Credentials and hashes
* Application content
* Configuration values

Common database systems include:

* MySQL
* Microsoft SQL Server
* Oracle
* PostgreSQL

From a security perspective, database access control is critical. A compromised application should not automatically mean full database compromise—but in practice, it often does.

---

## Additional Back End Components You May Encounter

Back end servers often include **supporting technologies**, such as:

* Hypervisors (virtualisation platforms)
* Containers (for example, Docker)
* Web Application Firewalls (WAFs)
* Monitoring and logging agents
* Backup and replication services

Each additional component increases complexity—and usually the attack surface.

---

## Common Back End Technology Stacks

Instead of building everything from scratch, many organisations deploy **predefined stacks**. These are well-known combinations of operating systems and software.

You will see these names frequently:

| Stack     | Components                                 |
| --------- | ------------------------------------------ |
| **LAMP**  | Linux, Apache, MySQL, PHP                  |
| **WAMP**  | Windows, Apache, MySQL, PHP                |
| **WINS**  | Windows, IIS, .NET, SQL Server             |
| **MAMP**  | macOS, Apache, MySQL, PHP                  |
| **XAMPP** | Cross-platform, Apache, MySQL, PHP or Perl |

Knowing these stacks helps you:

* Guess file locations
* Predict configuration defaults
* Anticipate common misconfigurations

Even when customised, many deployments still follow these patterns closely.

---

## Hardware and Performance Considerations

Behind all software is **hardware capacity**.

The available:

* CPU
* Memory
* Disk I/O
* Network bandwidth

Directly affects:

* Application stability
* Response time
* Ability to handle concurrent users

Poorly sized hardware often leads to:

* Timeouts
* Race conditions
* Inconsistent behaviour
* Misleading security symptoms

---

## Scaling Beyond a Single Server

Modern applications rarely rely on a single back end server.

Instead, you will often see:

* Multiple application servers behind a load balancer
* Dedicated database servers
* Redundant systems for availability
* Cloud-based virtual hosts

In these setups:

* No single server holds everything
* Compromise impact depends on segmentation quality
* Lateral movement becomes a real concern

As a tester, you should **never assume** that:

* One server equals the whole application
* Database access implies OS-level control
* Admin access on one node gives full infrastructure access

Sometimes it does. Often it does not.
