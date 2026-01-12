# IMAP and POP3

IMAP (Internet Message Access Protocol) and POP3 (Post Office Protocol v3) are protocols used to **access email on a mail server**.

* **IMAP** is designed for **online mailbox management**: it supports folders, synchronisation across multiple clients, and keeps messages on the server until explicitly deleted.
* **POP3** is designed for **simple retrieval**: it primarily supports listing, downloading, and deleting messages, and does not provide the same mailbox-management features as IMAP.

In practice, IMAP behaves more like a “remote mailbox filesystem”, while POP3 behaves more like “download my messages”.

---

## IMAP vs POP3 Behaviour

### IMAP

* Works best when you want **multiple devices** to stay in sync (laptop, phone, webmail).
* Emails typically **remain on the server** and are managed there (move, copy, delete, flag).
* Supports **folder hierarchies** (for example: INBOX, Sent, Important).

### POP3

* Focuses on **fetching messages** from the server.
* Usually used when a client wants to **download** mail and optionally delete it from the server.
* Does not natively provide the same folder structure and server-side browsing capabilities.

---

## Ports and Encryption

By default:

* **POP3:** 110 (plaintext), 995 (TLS/SSL)
* **IMAP:** 143 (plaintext), 993 (TLS/SSL)

Higher ports (993/995) are typically “implicit TLS” (encryption from the start).
On 110/143, encryption is often upgraded using `STARTTLS` if supported.

Without TLS, usernames, passwords, and content may be transmitted in **plain text**.

---

# Default Configuration Notes

IMAP/POP3 services (commonly **Dovecot**) have many configuration options. A practical way to understand them is to install:

* `dovecot-imapd`
* `dovecot-pop3d`

and then experiment locally.

Rather than deep-diving every setting, it is usually more useful to understand **how to interact with the services**, what commands exist, and what misconfigurations look like during enumeration.

---

# IMAP Commands (Common)

IMAP commands are tag-based. The tag (for example `1`) allows the client to match responses to requests.

| Command                         | Description                                       |
| ------------------------------- | ------------------------------------------------- |
| `1 LOGIN username password`     | Authenticates the user.                           |
| `1 LIST "" *`                   | Lists mailboxes/folders.                          |
| `1 CREATE "INBOX"`              | Creates a mailbox (name varies by server rules).  |
| `1 DELETE "INBOX"`              | Deletes a mailbox.                                |
| `1 RENAME "ToRead" "Important"` | Renames a mailbox.                                |
| `1 LSUB "" *`                   | Lists subscribed mailboxes.                       |
| `1 SELECT INBOX`                | Selects a mailbox for message access.             |
| `1 UNSELECT INBOX`              | Exits the selected mailbox.                       |
| `1 FETCH <ID> all`              | Retrieves message data for a specific message ID. |
| `1 CLOSE`                       | Removes messages flagged as `Deleted`.            |
| `1 LOGOUT`                      | Ends the IMAP session.                            |

---

# POP3 Commands (Common)

POP3 is simpler and command-driven without folder management.

| Command         | Description                                     |
| --------------- | ----------------------------------------------- |
| `USER username` | Identifies the user.                            |
| `PASS password` | Authenticates using a password.                 |
| `STAT`          | Shows mailbox statistics (count and size).      |
| `LIST`          | Lists messages with sizes.                      |
| `RETR id`       | Retrieves a specific message.                   |
| `DELE id`       | Marks a message for deletion.                   |
| `CAPA`          | Displays server capabilities.                   |
| `RSET`          | Resets the session state (undo deletion marks). |
| `QUIT`          | Ends the session.                               |

---

# Dangerous Settings

Misconfigurations in IMAP/POP3 can leak sensitive information or weaken authentication. Examples include over-verbose authentication logging or anonymous authentication mechanisms.

| Setting                   | Description                                                |
| ------------------------- | ---------------------------------------------------------- |
| `auth_debug`              | Enables authentication debug logging.                      |
| `auth_debug_passwords`    | Logs submitted passwords and schemes (high risk).          |
| `auth_verbose`            | Logs unsuccessful authentication attempts and reasons.     |
| `auth_verbose_passwords`  | Logs passwords used for authentication (may be truncated). |
| `auth_anonymous_username` | Username used for ANONYMOUS SASL mechanism.                |

In poorly secured environments, these can lead to credential exposure in logs or unintended access paths.

---

# Footprinting the Service

## Nmap Scan

A straightforward scan of common ports can reveal service type, version, TLS certificate details, and capabilities:

```bash
sudo nmap 10.129.14.128 -sV -p110,143,993,995 -sC
```

Typical output can include:

* Service banners (`Dovecot imapd`, `Dovecot pop3d`)
* Capability listings (for example, IMAP `IDLE`, POP3 `SASL`, `STLS`)
* TLS certificate metadata (CN, organisation, validity dates)

This is valuable for recon because certificates often reveal:

* **Hostname** (CN/SAN)
* **Organisation and location** (if present)
* Whether certificates are **self-signed** or issued by a CA

---

## cURL Interaction (IMAPS)

If you have valid credentials, you can interact with IMAPS using `curl`:

```bash
curl -k 'imaps://10.129.14.128' --user user:p4ssw0rd
```

Example folder listing output:

```text
* LIST (\HasNoChildren) "." Important
* LIST (\HasNoChildren) "." INBOX
```

Using verbose mode (`-v`) shows:

* TLS negotiation details
* Certificate subject/issuer
* Server capability banner
* Authentication steps

---

## OpenSSL Interaction (Encrypted POP3/IMAP)

To connect to the TLS-wrapped services directly:

### POP3S (995)

```bash
openssl s_client -connect 10.129.14.128:pop3s
```

### IMAPS (993)

```bash
openssl s_client -connect 10.129.14.128:imaps
```

This is useful for:

* Inspecting the certificate chain and parameters
* Confirming TLS versions/ciphers
* Reading raw server banners and capabilities

---

# Post-Login Usage

Once authenticated (IMAP or POP3), you can use the command sets above to:

* Enumerate folders/mailboxes (IMAP)
* List and retrieve messages (IMAP/POP3)
* Confirm server capabilities and behaviours (both)

A compromised mailbox can allow an attacker to:

* Read confidential internal messages
* Send mail as a legitimate user (depending on SMTP submission controls)
* Reset passwords elsewhere using intercepted reset links (secondary impact)

---

# Credential Testing Note

If you obtain credentials during testing (for example, from OSINT, password reuse, or other approved findings), they can be validated against IMAP/POP3 for mailbox access, assuming that is within scope and rules of engagement.
