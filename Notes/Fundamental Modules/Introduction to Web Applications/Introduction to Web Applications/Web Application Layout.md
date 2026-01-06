# Web Application Layout

No two web applications look the same behind the scenes. Each organisation designs and deploys applications based on its business goals, scale, budget, and operational constraints. As someone studying or testing web applications, your job is to **understand how an application is put together**, not just how it looks in the browser.

You should always think in terms of **structure, components, and relationships**. Once you understand those, weaknesses in design and security controls become much easier to spot 

---

## High-Level Web Application Layout

You can break down any web application layout into three main areas:

### 1. Web Application Infrastructure

This describes **where things live**.

It answers questions such as:

* Where is the database hosted?
* Is it on the same server as the application?
* Are there multiple application servers?
* Are services separated or tightly coupled?

Understanding infrastructure helps you reason about **impact**. A vulnerability on one server does not always mean full compromise of everything else.

---

### 2. Web Application Components

These are the **moving parts** the application interacts with.

At a minimum, you will always deal with:

* **Client components** (browser-side code)
* **Server components** (application logic)
* **Data storage** (databases or data services)

More complex applications also include:

* Third-party APIs
* Internal services
* Authentication providers
* Serverless functions

---

### 3. Web Application Architecture

Architecture describes **how components talk to each other**.

This includes:

* Trust boundaries
* Data flow
* Authentication paths
* Authorisation decisions

Most serious security issues are architectural, not purely coding mistakes.

---

## Common Infrastructure Models

You will usually encounter one of the following infrastructure designs.

---

### Client–Server Model

This is the most common setup.

* The **client** (browser) handles the UI
* The **server** hosts the application logic
* Communication happens over HTTP or HTTPS

Typical flow:

1. You visit a URL
2. The server sends the application interface
3. Your browser sends requests based on user actions
4. The server processes requests and returns responses

This learning platform itself follows this model: your browser is the client, and the Hack The Box servers handle everything else.

Even though this model is common, implementations vary widely.

![client-server-model](https://github.com/user-attachments/assets/cb54fcf3-6dc0-4aff-994b-12996d894b50)

---

### One Server Model

In this design:

* The web application
* The application logic
* The database

All run on **a single server**.

This setup is:

* Easy to deploy
* Cheap to maintain
* **High risk**

From a security point of view, this is an *all eggs in one basket* approach. If you compromise the web application, you usually compromise the database and everything else on the server.

Downtime is also catastrophic: if the server goes down, the entire application disappears.

![one-server-arch](https://github.com/user-attachments/assets/dc694399-7f22-437c-9694-054bb0dc2afb)

---

### Many Servers – One Database

Here, the database is separated from the application servers.

* One or more web servers
* A dedicated database server

Advantages:

* Better segmentation
* Reduced blast radius
* Easier scaling

If one web server is compromised, others may still be safe. Likewise, a database compromise does not automatically grant control over the web servers themselves.

However, segmentation alone is not enough. Access controls must still be enforced so each application can only access the data it truly needs.

-![many-server-one-db-arch](https://github.com/user-attachments/assets/b472ccaf-6a82-4d4a-84f2-fcb22914aec3)

--

### Many Servers – Many Databases

This model builds on separation even further.

* Each application has its own database
* Databases may be on separate servers
* Redundancy and failover are common

This design:

* Reduces lateral impact
* Improves resilience
* Is harder to deploy and manage

From a security perspective, this is often the **best option**, assuming access controls are implemented correctly. Load balancers and orchestration tools are commonly used here.

![many-server-many-db-arch](https://github.com/user-attachments/assets/2f19fcaf-f8c5-4c07-9b26-63eb4250bebd)

---

## Web Application Components (Detailed View)

Regardless of infrastructure, you can usually break an application down into the following components:

### Client

* Browser
* JavaScript
* HTML and CSS

Never trust anything here.

---

### Server

Usually includes:

* **Web server** (e.g. Apache, Nginx, IIS)
* **Application logic** (framework or custom code)
* **Database access layer**

This is where most critical vulnerabilities live.

---

### Services (Microservices)

These may include:

* Internal APIs
* Third-party integrations
* Payment processors
* Authentication services

Each service introduces:

* A new trust boundary
* A new attack surface

---

### Functions (Serverless)

Used heavily in cloud environments.

* Stateless execution
* Triggered by events
* Managed by cloud providers

Security responsibility still exists, even if infrastructure management is outsourced.

---

## Three-Tier Web Application Architecture

Most modern web applications follow a **three-tier architecture**.

### Presentation Layer

This is what you interact with:

* HTML
* JavaScript
* CSS

Returned to you by the server and rendered in the browser.

---

### Application Layer

This layer:

* Processes requests
* Enforces authorisation
* Applies business logic
* Validates input

Broken access control issues almost always live here.

---

### Data Layer

This layer:

* Stores data
* Retrieves records
* Interacts with databases

Injection flaws and data exposure issues often target this layer.

---

## Microservices Architecture

Microservices break applications into **small, independent services**, each focused on a single task.

Example for an online store:

* Registration service
* Search service
* Payment service
* Review service

Key properties:

* Stateless communication
* Independent deployment
* Language-agnostic components

Benefits:

* Faster development
* Better scalability
* Fault isolation

Trade-off:

* Increased complexity
* More communication paths
* More trust boundaries to secure

---

## Serverless Architecture

In serverless designs:

* You do not manage servers
* Code runs in isolated containers
* Cloud providers handle scaling and availability

Benefits:

* Rapid deployment
* Reduced operational overhead
* Automatic scaling

Security implication:

* You still own application logic security
* Misconfigured permissions can be devastating

---

## Architecture and Security

Not all vulnerabilities come from bad code.

Many serious issues are caused by **poor architectural decisions**, such as:

* Missing role-based access control (RBAC)
* Overly permissive service communication
* Shared databases without proper isolation
* Trusting internal services too much

For example:

* An application may be coded securely
* But flawed design allows regular users to access admin features

Fixing these issues usually requires **design changes**, not quick patches.

---

## Why This Matters to You

When you test a web application:

* Do not assume everything runs on one server
* Do not assume one database exists
* Do not assume compromise equals total access

Understanding layout and architecture helps you:

* Estimate real impact
* Avoid false assumptions
* Chain vulnerabilities intelligently

Security must be considered **at every stage of development**, and penetration testing should happen throughout the application lifecycle, not just at the end.

If you train yourself to think architecturally, web applications stop feeling like black boxes and start feeling predictable.
