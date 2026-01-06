# Introduction to Web Applications

Web applications are **interactive programs that run inside your browser** and follow a **client–server model**. What you see and interact with lives on the client side (HTML, CSS, JavaScript), while the logic, processing, and data storage live on the server side. This separation allows organisations to deploy powerful, centrally managed applications that are accessible from anywhere with an Internet connection 

From your perspective as a security practitioner, this split is important because **each side introduces its own risks, attack surface, and trust assumptions**.

![website_vs_webapps](https://github.com/user-attachments/assets/eadfdfa8-9622-472a-922d-4adbf2f19345)

---

## How Web Applications Are Structured

A typical web application is made up of two major parts:

### Client Side (Front End)

This is everything your browser handles:

* HTML (structure)
* CSS (presentation)
* JavaScript (behaviour)

The front end is responsible for how the application looks and how users interact with it, but it should **never be trusted**. Anything running in the browser can be inspected, modified, or replayed by an attacker.

### Server Side (Back End)

This is where the real work happens:

* Application logic
* Authentication and authorisation
* Database access
* File handling
* Business rules

The back end processes user input and decides what is allowed to happen. Most high-impact vulnerabilities originate here.

---

## Web Applications vs Traditional Websites

Older websites were mostly **static**. Everyone saw the same content, and nothing changed unless a developer manually edited the page. This model is often referred to as **Web 1.0**.

Modern web applications are very different:

* Content changes based on user interaction
* Pages are generated dynamically
* Applications respond in real time
* Users can create, modify, and delete data

This shift (often called **Web 2.0**) massively increases complexity—and complexity is where vulnerabilities thrive.

Key differences you should remember:

* Web applications are **interactive**
* They are **modular**
* They adapt to different devices and screen sizes
* They usually work across platforms without special optimisation

---

## Web Applications vs Native OS Applications

Unlike native operating system applications, web applications:

* Run inside a browser
* Do not require installation
* Are platform-independent
* Use server-side resources rather than local disk space

A major advantage is **version consistency**. Every user interacts with the same application version, and updates can be deployed instantly on the server without touching user systems.

However, native applications still have strengths:

* Faster performance
* Direct access to OS libraries
* Deeper hardware integration

To bridge this gap, you will increasingly encounter:

* **Hybrid applications**
* **Progressive Web Applications (PWAs)**

These combine web technologies with native capabilities, expanding both functionality and attack surface.

---

## Web Application Distribution Models

Web applications generally fall into two categories:

### Open-Source Applications

These are publicly available and highly customisable. Common examples include:

* WordPress
* OpenCart
* Joomla

Because the source code is available, attackers can study it extensively. Vulnerabilities often appear due to poor configuration or insecure plugins.

### Proprietary (Closed-Source) Applications

These are developed and maintained by vendors and usually offered via licences or subscriptions, such as:

* Wix
* Shopify
* DotNetNuke

Closed source does **not** mean secure. It only means attackers must reverse engineer behaviour instead of reading the code directly.

---

## Why Web Applications Are High-Risk

Web applications are:

* Internet-facing
* Globally accessible
* Constantly changing
* Heavily targeted by automated tools

Even small code changes can introduce serious vulnerabilities. Because web applications often connect to databases and internal systems, a single flaw can expose:

* User data
* Corporate information
* Credentials
* Internal infrastructure

This is why **continuous testing and secure development practices** are critical.

---

## Web Application Penetration Testing

Web application testing is no longer optional. Any organisation with external or internal web applications should test them regularly.

To test applications properly, you need to understand:

* How web applications are built
* How browsers and servers interact
* Where trust boundaries exist
* Which technologies are in use

A widely adopted methodology is the **OWASP Web Security Testing Guide**, which provides structured coverage of common risks.

---

## Typical Testing Approach

A practical assessment usually follows this flow:

1. **Front-End Review**

   * Inspect HTML, CSS, and JavaScript
   * Look for exposed logic, secrets, and client-side flaws
   * Identify issues such as XSS and sensitive data exposure

2. **Back-End Interaction Analysis**

   * Observe browser–server communication
   * Identify frameworks, languages, and services
   * Test input handling and business logic

3. **Unauthenticated Testing**

   * What can be accessed without logging in?
   * Can functionality be abused anonymously?

4. **Authenticated Testing**

   * What changes after login?
   * Can roles be bypassed or escalated?

Testing both perspectives is essential to avoid blind spots.

---

## Common Web Application Vulnerabilities (Real Impact)

Web application flaws often **chain together** and lead to serious compromise. Some examples you should keep in mind:

| Vulnerability            | Real-World Impact                                                             |
| ------------------------ | ----------------------------------------------------------------------------- |
| SQL Injection            | Extracting Active Directory usernames and launching password spraying attacks |
| File Inclusion           | Reading source code and escalating to remote code execution                   |
| Unrestricted File Upload | Uploading malicious code to fully compromise the web server                   |
| IDOR                     | Accessing or modifying another user’s data by changing object identifiers     |
| Broken Access Control    | Manipulating parameters (such as role IDs) to gain admin access               |

These are not theoretical. Variations of these issues appear repeatedly in real environments.

---

## Why Web Applications Matter So Much

It is increasingly rare to compromise an external host directly via a classic service exploit. Web applications have become the **primary entry point**.

A single vulnerable application can:

* Expose credentials
* Leak internal user lists
* Enable lateral movement
* Provide an initial foothold into the corporate network

If you are comfortable attacking networks and Active Directory but weak on web applications, you are missing one of the most important skill sets in modern security testing.
