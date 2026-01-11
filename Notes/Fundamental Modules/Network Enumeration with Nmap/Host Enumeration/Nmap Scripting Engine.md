## Nmap Scripting Engine (NSE)

The **Nmap Scripting Engine (NSE)** allows you to extend Nmap far beyond simple port and service detection. NSE scripts can interact directly with services, extract detailed information, and even perform lightweight vulnerability checks. As you enumerate a target, NSE helps you move from *what is running* to *how it behaves and what might be exploitable*.

From a workflow perspective, NSE scripts are most effective **after** you already know which ports and services are exposed.

---

## Running Scripts by Category

Scripts in NSE are grouped into categories such as `default`, `discovery`, `auth`, `safe`, `vuln`, and others. You can execute an entire category against a target using the following syntax:

```
sudo nmap <target> --script <category>
```

Example use cases:

* `discovery` → gather more information about services
* `auth` → test authentication mechanisms
* `vuln` → check for known vulnerabilities

---

## Running Specific Scripts

If you want fine-grained control, you can specify individual scripts instead of an entire category:

```
sudo nmap <target> --script <script-name>,<script-name>
```

This approach is preferred when you want to minimise noise and target a specific service.

---

## Example: SMTP Enumeration with NSE

Continuing with an SMTP service on port 25, you can extract useful information using targeted scripts.

```
sudo nmap 10.129.2.28 -p 25 --script banner,smtp-commands
```

### What this gives you

* **banner**
  Reveals the service banner, often exposing:

  * Mail server software
  * Hostname
  * Operating system hints

* **smtp-commands**
  Lists supported SMTP commands, which can indicate:

  * Whether user enumeration is possible (`VRFY`, `EXPN`)
  * Support for encryption (`STARTTLS`)
  * Message size limits and extensions

This type of output helps you understand how interactive and permissive the SMTP service is before attempting manual interaction.

---

## Aggressive Scanning (-A)

Nmap also provides an **aggressive scan mode**, which bundles several techniques into a single command:

```
sudo nmap 10.129.2.28 -p 80 -A
```

The `-A` option enables:

* Service version detection (`-sV`)
* Operating system detection (`-O`)
* Traceroute
* Default NSE scripts (`-sC`)

### When to use it

* During lab environments or controlled assessments
* When speed is more important than stealth

### When to avoid it

* In stealth-sensitive environments
* Where IDS/IPS detection is a concern

Aggressive scans generate significantly more traffic and logs.

---

## Interpreting Aggressive Scan Results

From a single aggressive scan, you can often identify:

* Web server software and version
* Underlying operating system (probabilistic)
* Web application frameworks or CMS platforms
* Page titles and application fingerprints
* Network distance (hop count)

This information is invaluable for prioritising follow-up attacks such as CMS exploitation or OS-specific privilege escalation.

---

## Vulnerability Scanning with NSE

NSE includes a **vuln** category designed to identify known vulnerabilities by querying services and matching versions against public databases.

Example:

```
sudo nmap 10.129.2.28 -p 80 -sV --script vuln
```

### What the vuln category does

* Enumerates application-specific endpoints
* Attempts basic vulnerability checks
* Correlates service versions with known CVEs
* Extracts usernames or admin paths when possible

For web services, this may reveal:

* CMS versions
* Login panels
* Known vulnerable plugins or components
* Publicly disclosed CVEs linked to the service

---

## Important Notes on NSE Vulnerability Scripts

* NSE vulnerability scripts **do not replace** dedicated vulnerability scanners
* Results should always be manually validated
* False positives and incomplete findings are common
* Scripts are best used as **guidance**, not proof

Treat NSE output as a prioritisation tool rather than a final verdict.
