# Extensions

Both Burp and ZAP support community-developed extensions that add new capabilities beyond what ships by default. These range from additional encoders and wordlists to specialised scanners for specific vulnerability classes.

***

## Burp BApp Store

Access it via `Extensions > BApp Store` inside Burp. Sort by Popularity to surface the most widely used and battle-tested extensions first. Some extensions are Pro-only, but the majority are available to Community users.

### Notable Extensions Worth Installing

| Extension | Purpose |
|-----------|---------|
| Decoder Improved | Extends Burp's built-in decoder with additional encoding and hashing options including MD5, SHA variants, and more |
| Active Scan++ | Adds additional active scan checks not included in Burp's default scanner |
| Autorize | Automates testing for broken access control by replaying requests with different privilege levels |
| CSRF Scanner | Identifies missing or weak CSRF protections across the application |
| JS Link Finder | Extracts endpoints and paths from JavaScript files, often revealing hidden API routes |
| Java Deserialization Scanner | Detects Java deserialization vulnerabilities |
| Software Version Reporter | Identifies software versions disclosed in responses for CVE correlation |
| Retire.JS | Flags JavaScript libraries with known vulnerabilities |
| CSP Auditor | Analyses Content Security Policy headers for weaknesses |
| J2EEScan | Adds active scan checks specific to Java EE applications |
| Backslash Powered Scanner | Extends server-side injection detection beyond standard payloads |
| Headers Analyzer | Reviews HTTP response headers for security misconfigurations |
| Error Message Checks | Detects verbose error messages that disclose stack traces or technology details |
| PHP Object Injection Check | Tests for PHP object injection vulnerabilities |
| AWS Security Checks | Identifies AWS-specific misconfigurations in traffic |

Some extensions require Jython (for Python-based extensions) or other runtimes not installed by default on Linux or macOS. If an extension fails to load, check its BApp Store page for dependency requirements.

***

## ZAP Marketplace

Access it by clicking "Manage Add-ons" and selecting the Marketplace tab. Add-ons are categorised by stability:

- **Release**: Stable and production-ready
- **Beta/Alpha**: Functional but may have rough edges

### Recommended Add-ons

The most immediately practical add-ons to install are FuzzDB Files and FuzzDB Offensive. These extend ZAP's built-in fuzzer with a comprehensive set of attack payloads and wordlists covering:

- OS command injection (`fuzzdb > attack > os-cmd-execution`)
- SQL injection
- XSS payloads
- Path traversal
- Authentication bypass strings

With FuzzDB installed, the File Fuzzers option in ZAP's fuzzer gains a substantially expanded payload library. Using the OS command injection wordlist against a vulnerable parameter, for example, will test a wide range of injection syntaxes that cover different operating systems and WAF bypass techniques simultaneously, which is especially useful when dealing with applications that have input filtering in place.

***

## Extending Your Workflow

Extensions and add-ons are what turn a general-purpose proxy into a specialised testing platform tailored to the application type you are assessing. A Java EE application warrants loading J2EEScan. A JavaScript-heavy SPA benefits from JS Link Finder and Retire.JS running passively. Cloud-hosted applications benefit from AWS Security Checks.

The practical habit to build is reviewing the BApp Store or ZAP Marketplace at the start of each new engagement and loading extensions relevant to the technology stack you are targeting. Knowing what extensions exist and when to reach for them is as important as knowing the tools themselves.

***

## Where to Go Next

These tools are foundational but their value compounds with hands-on practice against real targets. The natural progression after completing this module is:

- Apply Burp and ZAP workflows on HTB web application boxes and challenge labs
- Work through [PortSwigger's Web Security Academy](https://portswigger.net/web-security) labs, which are designed specifically around Burp usage
- Progress through HTB Academy's web attack modules covering SQLi, XSS, command injection, file upload bypass, and authentication attacks
- Use these tools alongside Nmap, ffuf, Gobuster, sqlmap, Hashcat, and Wireshark as part of a complete offensive toolkit

Burp and ZAP are not just web pentesting tools. Blue teamers use them to understand what attacks against their applications look like in transit, and developers use them to test their own applications before deployment. Proficiency with both covers a wide range of roles across offensive and defensive security work.
