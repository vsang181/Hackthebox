## DNS

The **Domain Name System (DNS)** functions as the internet’s translation layer, converting human-friendly domain names into machine-readable IP addresses. Just as a GPS converts a place name into geographic coordinates, DNS resolves domain names such as `www.example.com` into IP addresses like `192.0.2.1`, enabling systems to locate and communicate with one another.

Without DNS, users would need to memorise numerical IP addresses for every service they access, which would be impractical at scale. DNS abstracts this complexity, providing a hierarchical and distributed naming system that makes the modern internet usable.

---

## How DNS Works

When a user attempts to access a domain such as `www.example.com`, the following resolution process typically occurs:

1. **Local Cache Check**
   The client first checks its local DNS cache to see if the IP address is already known from a previous lookup.

2. **Recursive Resolver Query**
   If the address is not cached, the query is sent to a recursive DNS resolver, usually operated by the ISP or a public provider (e.g. Google DNS, Cloudflare).

3. **Root Name Server Query**
   The resolver queries a root name server. Root servers do not know the final IP but direct the resolver to the correct Top-Level Domain (TLD) name server.

4. **TLD Name Server Query**
   The TLD server (e.g. `.com`, `.org`) responds with the authoritative name server responsible for the target domain.

5. **Authoritative Name Server Query**
   The authoritative server returns the definitive DNS record (e.g. A or AAAA record) containing the IP address.

6. **Response and Caching**
   The resolver returns the IP address to the client and caches the result for the duration specified by the TTL (Time To Live).

7. **Connection Establishment**
   The client uses the resolved IP address to initiate a direct connection to the target server.

This hierarchical and cached approach allows DNS to scale efficiently while maintaining performance.

---

## The Hosts File

The **hosts file** provides a local, manual method of hostname-to-IP resolution that bypasses DNS entirely. It is processed before any DNS query is made.

**File locations:**

* Windows: `C:\Windows\System32\drivers\etc\hosts`
* Linux / macOS: `/etc/hosts`

**Syntax:**

```
<IP Address>    <Hostname> [<Alias> ...]
```

**Examples:**

```
127.0.0.1       localhost
192.168.1.10    devserver.local
```

### Common Use Cases

* **Local development**

  ```
  127.0.0.1       myapp.local
  ```

* **Testing connectivity**

  ```
  192.168.1.20    testserver.local
  ```

* **Blocking domains**

  ```
  0.0.0.0         unwanted-site.com
  ```

Changes to the hosts file take effect immediately and require administrative or root privileges to edit.

---

## DNS Zones and Zone Files

A **DNS zone** is an administrative portion of the DNS namespace managed by a specific entity. For example, `example.com` and its subdomains (`mail.example.com`, `www.example.com`) typically belong to the same zone.

A **zone file** defines all DNS records for a zone and is stored on an authoritative DNS server.

### Example Zone File

```
$TTL 3600
@       IN SOA   ns1.example.com. admin.example.com. (
                2024060401
                3600
                900
                604800
                86400 )

@       IN NS    ns1.example.com.
@       IN NS    ns2.example.com.
@       IN MX 10 mail.example.com.
www     IN A     192.0.2.1
mail    IN A     198.51.100.1
ftp     IN CNAME www.example.com.
```

This file defines:

* Authoritative name servers (NS)
* Mail servers (MX)
* Host-to-IP mappings (A)
* Aliases (CNAME)

---

## Key DNS Concepts

| Concept              | Description                  | Example                 |
| -------------------- | ---------------------------- | ----------------------- |
| Domain Name          | Human-readable identifier    | `www.example.com`       |
| IP Address           | Numerical network identifier | `192.0.2.1`             |
| DNS Resolver         | Performs recursive lookups   | ISP resolver, `8.8.8.8` |
| Root Name Server     | Top of DNS hierarchy         | `a.root-servers.net`    |
| TLD Name Server      | Manages a TLD                | `.com`, `.org`          |
| Authoritative Server | Holds final records          | Hosting provider DNS    |
| DNS Records          | Stored DNS data types        | A, AAAA, MX, TXT        |

---

## Common DNS Record Types

| Record | Full Name          | Purpose                 | Example                                                    |
| ------ | ------------------ | ----------------------- | ---------------------------------------------------------- |
| A      | Address            | IPv4 mapping            | `www.example.com. IN A 192.0.2.1`                          |
| AAAA   | IPv6 Address       | IPv6 mapping            | `www.example.com. IN AAAA 2001:db8::1`                     |
| CNAME  | Canonical Name     | Alias                   | `blog.example.com. IN CNAME web.example.net.`              |
| MX     | Mail Exchange      | Mail routing            | `example.com. IN MX 10 mail.example.com.`                  |
| NS     | Name Server        | Zone delegation         | `example.com. IN NS ns1.example.com.`                      |
| TXT    | Text               | Metadata / verification | `example.com. IN TXT "v=spf1 mx -all"`                     |
| SOA    | Start of Authority | Zone metadata           | Serial, timers                                             |
| SRV    | Service            | Service discovery       | `_sip._tcp.example.com. IN SRV 10 5 5060 sip.example.com.` |
| PTR    | Pointer            | Reverse DNS             | `1.2.0.192.in-addr.arpa. IN PTR www.example.com.`          |

The `IN` class denotes the **Internet** protocol suite and is standard in modern DNS records.

---

## Why DNS Matters for Web Reconnaissance

DNS is a critical reconnaissance surface and often leaks valuable infrastructure details:

* **Asset Discovery**
  Subdomains, mail servers, VPN endpoints, and internal naming conventions can be exposed via DNS records.

* **Infrastructure Mapping**
  NS and MX records reveal hosting providers, third-party services, and external dependencies.

* **Attack Surface Expansion**
  Forgotten or deprecated subdomains (`old.example.com`, `dev.example.com`) often point to vulnerable systems.

* **Technology & Tooling Insights**
  TXT records can reveal SaaS usage (SPF, DKIM, password managers, CI/CD platforms).

* **Change Detection**
  Monitoring DNS over time can highlight newly deployed services or entry points.

DNS enumeration is low-noise, high-value, and should always be performed early during web reconnaissance to guide deeper testing and prioritisation.
