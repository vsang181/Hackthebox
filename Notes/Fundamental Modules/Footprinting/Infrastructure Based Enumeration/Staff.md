## Staff

Identifying employees and their roles across public platforms can reveal a surprising amount about a company’s infrastructure, technology stack, and internal maturity. Staff OSINT is not limited to names and titles. It can help us map likely systems, tooling, programming languages, cloud platforms, and security controls based on what people build, maintain, and talk about publicly.

Employees can often be found on professional networks such as LinkedIn or Xing. Beyond employee profiles, company job postings are especially useful because they often describe required skills in a way that indirectly documents the organisation’s architecture and engineering practices.

---

### Job Postings as a Technology and Process Indicator

A single technical job advert can provide strong hints about:

* Preferred programming languages and runtimes (e.g., Java, C#, Python)
* Backend and web frameworks (e.g., Django, Flask, Spring, ASP.NET)
* Database technologies (e.g., PostgreSQL, MySQL, Oracle)
* DevOps and delivery practices (CI/CD, version control)
* Platform tooling (Atlassian, Git hosting, ticketing systems)
* Deployment patterns (microservices, containers, orchestration)

Example excerpt from a job post:

**Required Skills/Knowledge/Experience:**

* Experience with one or more object-oriented languages (e.g., Java, C#, C++)
* Experience with one or more scripting languages (e.g., Python, Ruby, PHP, Perl)
* Experience using SQL databases (e.g., PostgreSQL, MySQL, SQL Server, Oracle)
* Experience using ORM frameworks (e.g., SQLAlchemy, Hibernate, Entity Framework)
* Experience using Web frameworks (e.g., Flask, Django, Spring, ASP.NET MVC)
* Proficient with unit testing and test frameworks (e.g., pytest, JUnit, NUnit, xUnit)
* Service-Oriented Architecture (SOA) / microservices and RESTful API design
* Familiarity with Agile development processes and Continuous Integration environments
* Experience with version control systems (e.g., Git, SVN, Mercurial, Perforce)

**Desired Skills/Knowledge/Experience:**

* Experience with Atlassian suite (Confluence, Jira, Bitbucket)
* Software security
* Containerisation and orchestration (Docker, Kubernetes)
* Redis
* NumPy

From requirements like these, we can infer that the organisation likely has:

* One or more web applications and APIs (REST is explicitly mentioned)
* CI pipelines and automated testing
* Multiple environments (dev, staging, production) that support iterative delivery
* A codebase structured around ORM and framework conventions
* Atlassian tooling that may expose public artefacts (misconfigured Confluence spaces, Jira project keys, Bitbucket repos)

These insights help shape later enumeration. For example, if Django is used, we might later look for common Django artefacts and deployment patterns such as:

* `/static/` and `/media/` handling
* Debug misconfigurations (`DEBUG=True`)
* Exposed admin panels (`/admin/`)
* Leaked `SECRET_KEY` values in repositories or environment files
* Incorrectly configured ALLOWED_HOSTS, CORS, or CSRF settings
* Default or predictable file and folder naming conventions in deployments

---

### Employee Profiles and Public Technical Footprints

Employee profiles often contain more detailed hints than job postings, including:

* Technologies they actively use (frameworks, languages, cloud platforms)
* The type of systems they work on (mobile apps, internal tooling, data pipelines)
* Links to GitHub, personal blogs, or open-source contributions
* Conference talks, write-ups, or posts about current work

Example signals found in profiles might include:

* Frontend frameworks and modern web components (React, Svelte, AngularJS)
* References to standards or specifications (W3C, web components)
* GitHub links to repositories that may include code samples or tooling

This matters because personal repositories and snippets sometimes accidentally include sensitive data, such as:

* Personal or corporate email addresses
* Internal hostnames or environment names
* API keys, tokens, or service credentials
* Hardcoded secrets (for example, JWT secrets or example tokens reused in production)
* Configuration patterns that mirror internal deployments

Even when the code is not directly exploitable, it often reveals naming conventions and architecture decisions that can be re-used for targeted enumeration later (for example, repo naming, service naming, or directory structure).

---

### Why Staff OSINT Matters for Security Assessment

Public employee data can help estimate:

* The organisation’s defensive maturity (presence of security engineers, SOC analysts, AppSec roles)
* Likely security tooling (SIEM platforms, EDR vendors, WAF providers)
* Common authentication patterns (SSO adoption, cloud identity providers, MFA references)
* Internal engineering practices (code review, CI usage, testing culture)

When the goal is to infer infrastructure and technology choices, it is usually most effective to focus on technical staff across:

* Software engineering (backend, frontend, platform)
* DevOps / SRE
* Cloud engineering
* Security engineering / SOC / incident response
* IT operations and network engineering

The more clearly an employee’s role aligns with building or defending systems, the more useful their public footprint becomes for understanding the organisation’s stack and likely attack surface.
