## Digging DNS

Having established a solid understanding of DNS fundamentals and record types, this section focuses on **practical DNS reconnaissance**. DNS is one of the most valuable low-noise information sources during web reconnaissance, and knowing how to interrogate it effectively is essential.

---

## DNS Reconnaissance Tools

DNS reconnaissance relies on tools designed to query DNS servers and extract meaningful information about a target’s infrastructure.

| Tool                 | Key Features                                                                      | Typical Use Cases                                               |
| -------------------- | --------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| **dig**              | Highly versatile DNS query tool supporting most record types with detailed output | Manual DNS queries, zone transfer testing, deep record analysis |
| **nslookup**         | Simple DNS query utility                                                          | Quick resolution checks (A, AAAA, MX)                           |
| **host**             | Minimalistic DNS lookup tool                                                      | Fast lookups with concise output                                |
| **dnsenum**          | Automated enumeration with wordlists and brute forcing                            | Subdomain discovery, zone transfer checks                       |
| **fierce**           | Recursive subdomain enumeration with wildcard detection                           | Identifying hidden or forgotten subdomains                      |
| **dnsrecon**         | Multi-technique DNS reconnaissance with structured output                         | Comprehensive DNS mapping                                       |
| **theHarvester**     | OSINT-focused reconnaissance tool                                                 | Collecting emails, hosts, and infrastructure data               |
| **Online DNS tools** | Browser-based DNS queries                                                         | Quick lookups when CLI access is unavailable                    |

---

## The Domain Information Groper (dig)

`dig` (Domain Information Groper) is the most commonly used DNS reconnaissance tool due to its flexibility and depth of output.

### Common `dig` Commands

| Command                         | Description                                      |
| ------------------------------- | ------------------------------------------------ |
| `dig domain.com`                | Default A-record lookup                          |
| `dig domain.com A`              | Retrieve IPv4 address                            |
| `dig domain.com AAAA`           | Retrieve IPv6 address                            |
| `dig domain.com MX`             | Identify mail servers                            |
| `dig domain.com NS`             | Identify authoritative name servers              |
| `dig domain.com TXT`            | Retrieve TXT records                             |
| `dig domain.com CNAME`          | Retrieve canonical name                          |
| `dig domain.com SOA`            | Retrieve SOA record                              |
| `dig @1.1.1.1 domain.com`       | Query a specific DNS server                      |
| `dig +trace domain.com`         | Show full resolution path                        |
| `dig -x 192.168.1.1`            | Reverse DNS lookup                               |
| `dig +short domain.com`         | Minimal output                                   |
| `dig +noall +answer domain.com` | Answer section only                              |
| `dig domain.com ANY`            | Request all records (often blocked per RFC 8482) |

> **Caution**
> Excessive DNS queries can be logged, rate-limited, or blocked. Always stay within scope and obtain proper authorization before performing extensive enumeration.

---

## Practical Example: Querying DNS with `dig`

### Basic Query

```
dig google.com
```

This command performs a default A-record lookup for `google.com`.

### Key Sections of the Output

#### 1. Header

* **opcode: QUERY** – Standard DNS query
* **status: NOERROR** – Query succeeded
* **flags**:

  * `qr` – This is a response
  * `rd` – Recursion requested
  * `ad` – Authentic data (DNSSEC-validated, if applicable)
* Section counts indicate how many records appear in each part of the response.

A warning such as:

```
recursion requested but not available
```

means the queried server does not support recursion.

---

#### 2. Question Section

```
google.com. IN A
```

This asks for the IPv4 address associated with `google.com`.

---

#### 3. Answer Section

```
google.com. 0 IN A 142.251.47.142
```

* **A record** mapping `google.com` to an IPv4 address
* **TTL (0)** indicates how long the result may be cached

---

#### 4. Footer

* **Query time** – How long the lookup took
* **SERVER** – DNS server used to answer the query
* **WHEN** – Timestamp of the query
* **MSG SIZE** – Size of the DNS response

---

## Simplified Output with `+short`

If only the resolved values are required, use:

```
dig +short hackthebox.com
```

Example output:

```
104.18.20.126
104.18.21.126
```

This is ideal for scripting, automation, or rapid reconnaissance.
