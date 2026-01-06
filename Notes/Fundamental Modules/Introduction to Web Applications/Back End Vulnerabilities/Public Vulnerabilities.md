# Public Vulnerabilities

Some of the **most critical vulnerabilities** in a web application are those that can be exploited **externally**, without any prior access to the back-end server. These are the types of issues targeted during **external penetration tests**, where the attacker interacts only with what is exposed over the network.

Such vulnerabilities are usually introduced through:

* Coding mistakes in back-end components
* Unsafe use of third-party software
* Poor patching and update practices

The impact of these issues can range from simple information disclosure to **full remote compromise of the back-end server**.

---

## Public CVEs

Many organisations deploy web applications that are:

* Open source
* Commercial off-the-shelf
* Widely used across the internet

Because of this exposure, these applications are regularly analysed by security researchers worldwide. When vulnerabilities are discovered, they are often:

* Reported publicly
* Assigned a **CVE (Common Vulnerabilities and Exposures)** identifier
* Given a severity score

In most cases, a patch is released, and the vulnerability details become publicly available.

Security researchers and penetration testers frequently create **proof-of-concept exploits** to demonstrate how a vulnerability can be abused. These are often shared publicly for educational and testing purposes. As a result, **searching for public exploits is usually the first step** when assessing a web application.

---

### Identifying the Application Version

Before searching for public vulnerabilities, you must first identify the **exact version** of the target application.

You can often find version information in:

* Application source code
* Comments in HTML pages
* Configuration files
* Publicly accessible version files (for example, `version.php`)

For open-source applications, you can inspect the official repository to see where version information is displayed and then compare it with the target system.

Once the version is confirmed, you can search for known vulnerabilities using:

* General search engines
* Public exploit databases such as Exploit DB, Rapid7, or Vulnerability Lab

When reviewing results, you should prioritise:

* Vulnerabilities with **CVSS scores between 8 and 10**
* Vulnerabilities that lead to **Remote Code Execution (RCE)**

That said, other vulnerability types should still be considered if no critical exploits are available.

---

### Third-Party Components

Public vulnerabilities are not limited to the core application itself.

If the application relies on:

* Plugins
* Themes
* External libraries
* Framework components

You should also search for vulnerabilities affecting those components. In many real-world compromises, the weakest point is not the main application, but an outdated or poorly maintained dependency.

---

## Common Vulnerability Scoring System (CVSS)

The **Common Vulnerability Scoring System (CVSS)** is an industry-standard method for rating the severity of security vulnerabilities. It is widely used by organisations and governments to:

* Assess risk
* Prioritise remediation
* Communicate impact consistently

CVSS scores are calculated using several metric groups:

* **Base**
* **Temporal**
* **Environmental**

The Base metrics produce a score from **0 to 10**, which represents the inherent severity of a vulnerability. Temporal and Environmental metrics are then used to adjust this score based on factors such as exploit availability and organisational impact.

The **National Vulnerability Database (NVD)** provides CVSS scores for most publicly disclosed vulnerabilities. At present, the NVD publishes **Base scores only**, as Temporal and Environmental factors vary between organisations.

Two scoring systems are currently in use:

* **CVSS v2**
* **CVSS v3**

They differ in how severity levels and metrics are defined.

---

### CVSS Severity Ratings

#### CVSS v2.0

| Severity | Base Score |
| -------- | ---------- |
| Low      | 0.0 – 3.9  |
| Medium   | 4.0 – 6.9  |
| High     | 7.0 – 10.0 |

#### CVSS v3.0

| Severity | Base Score |
| -------- | ---------- |
| None     | 0.0        |
| Low      | 0.1 – 3.9  |
| Medium   | 4.0 – 6.9  |
| High     | 7.0 – 8.9  |
| Critical | 9.0 – 10.0 |

The NVD provides interactive CVSS calculators for both v2 and v3, which allow you to:

* Adjust metrics
* Model real-world impact
* Fine-tune severity for your environment

Experimenting with these calculators is an excellent way to understand how different factors influence risk.

---

## Back-End Server Vulnerabilities

In addition to application-level vulnerabilities, you should also consider **back-end infrastructure vulnerabilities**, including:

* Web servers
* Operating systems
* Database servers

The most critical back-end vulnerabilities are often found in **web servers**, since they are directly exposed over the network. A well-known example is **Shellshock**, which affected Apache-based systems and allowed remote command execution via crafted HTTP requests.

Vulnerabilities in:

* Operating systems
* Databases

Are usually exploited **after initial access** has been gained. These are commonly used during:

* Privilege escalation
* Lateral movement
* Internal penetration tests

Although these vulnerabilities may not always be exploitable directly from the internet, they are still **high impact** and must be patched. Once an attacker gains a foothold, unpatched internal vulnerabilities often determine how far the compromise can spread.
