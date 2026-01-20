## Utilising WHOIS

WHOIS data becomes most valuable when interpreted in context. Below are practical scenarios that demonstrate how WHOIS can support investigations, threat analysis, and intelligence reporting.

---

### Scenario 1: Phishing Investigation

An email security gateway flags a suspicious message sent to multiple employees. The email impersonates a bank and urges recipients to click a link to “verify” account details. A WHOIS lookup is performed on the linked domain.

Key WHOIS findings:

* **Registration Date**
  The domain was registered only a few days ago.
* **Registrant Information**
  Registrant details are hidden behind a privacy protection service.
* **Name Servers**
  DNS servers are linked to a hosting provider frequently associated with malicious or “bulletproof” infrastructure.

These indicators strongly suggest a phishing campaign. Recently registered domains with hidden ownership and suspicious name servers are commonly used for short-lived attacks. Based on this information, the domain can be blocked, employees alerted, and the hosting provider investigated for related malicious infrastructure.

---

### Scenario 2: Malware Analysis

A malware sample is observed communicating with a remote command-and-control (C2) domain. WHOIS is used to gain insight into the attacker’s infrastructure.

Key WHOIS findings:

* **Registrant**
  The domain is registered to an individual using a free, anonymity-friendly email provider.
* **Geographical Location**
  The registrant address points to a region frequently associated with cybercrime activity.
* **Registrar**
  The registrar has a history of weak abuse handling and delayed takedowns.

These indicators suggest the C2 domain is either hosted on compromised infrastructure or intentionally placed with a provider tolerant of abuse. WHOIS data helps attribute infrastructure patterns and supports takedown or notification efforts.

---

### Scenario 3: Threat Intelligence Reporting

A security team tracks a threat actor targeting financial institutions. WHOIS records are collected for domains used across multiple campaigns.

Observed patterns include:

* **Registration Timing**
  Domains are registered in clusters shortly before attack windows.
* **Registrant Aliases**
  Multiple fake identities and recycled contact details are used.
* **Shared Name Servers**
  Reuse of DNS infrastructure indicates common backend management.
* **Lifecycle Behaviour**
  Domains are frequently abandoned or taken down shortly after campaigns conclude.

These insights help build a profile of the threat actor’s **tactics, techniques, and procedures (TTPs)** and generate actionable indicators of compromise (IOCs) for defensive teams.

---

## Using WHOIS on Linux

Before performing WHOIS queries, ensure the tool is installed.

### Installation

```
sudo apt update
sudo apt install whois -y
```

### Basic WHOIS Query

```
whois facebook.com
```

Example output (truncated):

```
Domain Name: FACEBOOK.COM
Registrar: RegistrarSafe, LLC
Creation Date: 1997-03-29
Registry Expiry Date: 2033-03-30
Name Server: A.NS.FACEBOOK.COM
Name Server: B.NS.FACEBOOK.COM
```

---

## Interpreting WHOIS Results (Example: facebook.com)

### Domain Registration

* **Registrar**: RegistrarSafe, LLC
* **Creation Date**: 1997-03-29
* **Expiry Date**: 2033-03-30

A long registration history and distant expiry date typically indicate a legitimate, well-established domain.

### Domain Ownership

* **Organisation**: Meta Platforms, Inc.
* **Contact**: Domain Admin

Large organisations often use generic role-based contacts rather than individual employees.

### Domain Status Flags

* `clientDeleteProhibited`
* `clientTransferProhibited`
* `clientUpdateProhibited`
* `serverDeleteProhibited`
* `serverTransferProhibited`
* `serverUpdateProhibited`

These flags indicate strong protections against unauthorised domain changes at both registrar and registry levels.

### Name Servers

* `A.NS.FACEBOOK.COM`
* `B.NS.FACEBOOK.COM`
* `C.NS.FACEBOOK.COM`
* `D.NS.FACEBOOK.COM`

Using internally managed name servers suggests full control over DNS infrastructure, which is common for large enterprises.
