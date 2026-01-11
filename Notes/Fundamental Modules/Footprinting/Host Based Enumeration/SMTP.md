# SMTP

Simple Mail Transfer Protocol (SMTP) is the standard protocol used to **send email** across IP networks. SMTP is used in two common paths:

* Between a **Mail User Agent (MUA)** (for example, an email client or application) and a **mail submission server**
* Between two **SMTP servers** for server-to-server mail delivery

Although SMTP is traditionally described as a client-server protocol, it is normal for an SMTP server to behave like a client when it forwards mail to another server.

SMTP is typically paired with **IMAP** or **POP3**, which are used for **retrieving** emails from a mailbox.

---

## Ports and Transport

By default, SMTP servers accept connections on **TCP/25** for server-to-server delivery. Modern environments also commonly expose:

* **TCP/587 (Submission)**: Used by authenticated users/applications submitting outbound mail, usually upgraded to TLS with `STARTTLS`.
* **TCP/465 (SMTPS)**: Implicit TLS (TLS from the start), historically used and still widely supported for legacy/compatibility.

### TLS Modes

SMTP can operate in plaintext or encrypted mode:

* **Plain SMTP**: Commands and content are transmitted in clear text.
* **STARTTLS (explicit TLS)**: Begins in plaintext, then upgrades to TLS after `STARTTLS`.
* **SMTPS (implicit TLS)**: TLS is negotiated immediately (typically on 465).

If authentication is used without TLS, credentials can be exposed via sniffing on the network path.

---

## Mail Flow Components (MUA → Mailbox)

Email delivery involves multiple logical components:

* **MUA (Mail User Agent)**: The client creating/sending the email.
* **MSA (Mail Submission Agent)**: Validates that the sender is allowed to submit mail (often runs on 587).
* **MTA (Mail Transfer Agent)**: Relays mail between servers and performs routing via DNS (MX lookups).
* **MDA (Mail Delivery Agent)**: Delivers mail into the recipient’s mailbox/storage.
* **Mailbox access**: POP3/IMAP used by the recipient to fetch mail.

Typical flow:

`Client (MUA) → Submission Agent (MSA) → Mail Transfer Agent (MTA) → Mail Delivery Agent (MDA) → Mailbox (POP3/IMAP)`

A misconfigured MTA can behave as an **open relay**, allowing unauthenticated third parties to send email through it.

---

## Protocol Limitations and Abuse

SMTP has two inherent issues that matter both operationally and during security testing:

1. **No guaranteed delivery confirmation**
   SMTP can report acceptance (queued) or failure codes, but it does not guarantee that an email was actually delivered to the user’s inbox. Delivery Status Notifications (DSN) exist, but behaviour varies widely.

2. **Sender identity is not inherently trustworthy**
   SMTP allows the client to provide arbitrary `MAIL FROM` values. Without additional controls, this enables spoofing. Modern defences include:

   * **SPF** (Sender Policy Framework)
   * **DKIM** (DomainKeys Identified Mail)
   * **DMARC** (policy layer that ties SPF/DKIM alignment to enforcement)

These reduce spoofing success, but they do not eliminate phishing (for example, lookalike domains, compromised accounts, or abused legitimate services).

---

## ESMTP and STARTTLS

Extended SMTP (ESMTP) is effectively “SMTP with extensions”. In practice, when people say “SMTP”, they usually mean ESMTP.

After connecting, the client typically sends `EHLO` (instead of legacy `HELO`) to request the server’s extension list. If `STARTTLS` is offered, the client can upgrade the session to TLS and then safely perform authentication such as `AUTH PLAIN` or other mechanisms supported by the server.

---

# Default Configuration (Postfix Example)

SMTP servers are highly configurable. For Postfix, the main configuration file is typically:

* `/etc/postfix/main.cf`

Example (filtered) configuration output:

```bash
cat /etc/postfix/main.cf | grep -v "#" | sed -r "/^\s*$/d"
```

```text
smtpd_banner = ESMTP Server 
biff = no
append_dot_mydomain = no
readme_directory = no
compatibility_level = 2
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
myhostname = mail1.inlanefreight.htb
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
smtp_generic_maps = hash:/etc/postfix/generic
mydestination = $myhostname, localhost 
masquerade_domains = $myhostname
mynetworks = 127.0.0.0/8 10.129.0.0/16
mailbox_size_limit = 0
recipient_delimiter = +
smtp_bind_address = 0.0.0.0
inet_protocols = ipv4
smtpd_helo_restrictions = reject_invalid_hostname
home_mailbox = /home/postfix
```

Two settings worth highlighting during enumeration:

* `myhostname`: often reveals naming conventions (mail hostnames, internal domains)
* `mynetworks`: defines which source networks are treated as “trusted” (critical for relay behaviour)

---

# Common SMTP Commands

SMTP uses command-and-response interaction with numeric status codes.

| Command      | Description                                                                   |
| ------------ | ----------------------------------------------------------------------------- |
| `AUTH PLAIN` | Auth extension used to authenticate the client (usually only safe after TLS). |
| `HELO`       | Legacy greeting (starts session with client hostname).                        |
| `EHLO`       | Extended greeting; requests advertised ESMTP extensions.                      |
| `MAIL FROM:` | Sets envelope sender (used for routing/bounces).                              |
| `RCPT TO:`   | Sets envelope recipient.                                                      |
| `DATA`       | Begins sending the email content (headers + body).                            |
| `RSET`       | Resets the current mail transaction without closing the connection.           |
| `VRFY`       | Attempts to verify whether a mailbox/user exists.                             |
| `EXPN`       | Expands a mailing list / alias (often disabled).                              |
| `NOOP`       | Keepalive / request a response without action.                                |
| `QUIT`       | Ends the session.                                                             |

---

# Service Interaction

## Telnet: HELO / EHLO

You can use `telnet` to speak SMTP directly:

```bash
telnet 10.129.14.128 25
```

Example interaction:

```text
220 ESMTP Server 

HELO mail1.inlanefreight.htb
250 mail1.inlanefreight.htb

EHLO mail1
250-mail1.inlanefreight.htb
250-PIPELINING
250-SIZE 10240000
250-ETRN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING
```

From `EHLO` output you can infer capability and sometimes security posture, for example:

* `STARTTLS` present or absent
* Authentication mechanisms (`AUTH`)
* Message size constraints (`SIZE`)
* Delivery status support (`DSN`)

---

## User Enumeration with VRFY (Unreliable)

`VRFY` can sometimes be used to check if a user exists, but it is frequently disabled or deliberately “lied about”.

Example:

```text
VRFY root
252 2.0.0 root

VRFY testuser
252 2.0.0 testuser
```

A `252` response can mean the server will accept the message and attempt delivery, but it does **not** confirm existence. Many servers respond this way to reduce enumeration value.

Because of this, enumeration should not rely on a single signal. Always validate findings using multiple approaches and corroborate with other evidence (for example, SMTP errors during `RCPT TO`, directory leaks, password reset flows, or corporate naming patterns).

---

## Sending an Email Manually

SMTP can be used to build and send a message from the command line. This is useful for understanding how envelopes and headers differ:

* **Envelope** (`MAIL FROM`, `RCPT TO`) controls routing
* **Headers** (`From:`, `To:`, `Subject:`) are part of message content

Example:

```text
EHLO inlanefreight.htb
250-mail1.inlanefreight.htb
250-PIPELINING
250-SIZE 10240000
250-ETRN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250-DSN
250-SMTPUTF8
250 CHUNKING

MAIL FROM:<cry0l1t3@inlanefreight.htb>
250 2.1.0 Ok

RCPT TO:<mrb3n@inlanefreight.htb> NOTIFY=success,failure
250 2.1.5 Ok

DATA
354 End data with <CR><LF>.<CR><LF>

From: <cry0l1t3@inlanefreight.htb>
To: <mrb3n@inlanefreight.htb>
Subject: DB
Date: Tue, 28 Sept 2021 16:32:51 +0200

Hey man, I am trying to access our XY-DB but the creds don't work.
Did you make any changes there?
.
250 2.0.0 Ok: queued as 6E1CF1681AB

QUIT
221 2.0.0 Bye
```

---

## Email Headers (Recon Value)

Email headers can leak a lot of useful information, including:

* Internal hostnames and relay chain (`Received:` headers)
* Mail gateway products and filtering layers
* Timestamps and time zones
* Message IDs and formatting details
* Sometimes internal IPs (less common in modern hardened setups)

Header format and semantics are defined in RFC 5322, and delivery/relay behaviour is influenced by many other RFCs and vendor-specific implementations.

---

# Dangerous Settings

## Open Relay

An “open relay” is an SMTP server that will forward mail to external recipients without proper authentication or source restrictions. This is commonly caused by overly permissive trust settings.

Example dangerous Postfix config:

```text
mynetworks = 0.0.0.0/0
```

If a server treats everyone as trusted, it can be abused to:

* Send spam and phishing at scale
* Spoof sender identities (where downstream controls allow)
* Damage domain/IP reputation (blacklisting)
* Support multi-stage social engineering campaigns

Modern SMTP setups should restrict relaying to authenticated users and defined internal networks, and enforce TLS where applicable.

---

# Footprinting the Service

## Nmap SMTP Capabilities

Nmap’s default SMTP scripts can enumerate server capabilities by issuing `EHLO`:

```bash
sudo nmap 10.129.14.128 -sC -sV -p25
```

Example result:

```text
25/tcp open  smtp    Postfix smtpd
|_smtp-commands: mail1.inlanefreight.htb, PIPELINING, SIZE 10240000, VRFY, ETRN, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING,
```

---

## Nmap Open Relay Check

Nmap includes an `smtp-open-relay` script that attempts multiple relay patterns to determine whether the server is an open relay:

```bash
sudo nmap 10.129.14.128 -p25 --script smtp-open-relay -v
```

Example output (truncated):

```text
| smtp-open-relay: Server is an open relay (16/16 tests)
|  MAIL FROM:<> -> RCPT TO:<relaytest@nmap.scanme.org>
|  MAIL FROM:<antispam@nmap.scanme.org> -> RCPT TO:<relaytest@nmap.scanme.org>
|  ...
```

Even if a tool reports “open relay”, validate manually (and only within scope) because relay behaviour can differ based on:

* Source IP reputation / allowlists
* Recipient domains
* Sender domains
* Authentication requirements
* TLS policies and MSA vs MTA separation
