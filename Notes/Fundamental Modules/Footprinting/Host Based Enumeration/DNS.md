# DNS

Domain Name System (DNS) is a core component of how the Internet works. Humans prefer names like `academy.hackthebox.com`, but networks route traffic using IP addresses. DNS acts as the translation layer that resolves **hostnames → IP addresses** (and sometimes IP addresses back to names).

DNS does not have a single “central database”. Instead, it is a distributed hierarchy of servers, similar to a library system with many phone books spread across the world. A client asks questions, and DNS servers either answer directly (if they are authoritative) or help find the answer through recursion and caching.

---

## DNS Server Types

DNS resolution involves multiple server roles. The exact architecture depends on the environment (home router, enterprise DNS, public resolvers, etc.), but the main categories are:

* DNS root server
* Authoritative name server
* Non-authoritative name server
* Caching server
* Forwarding server
* Resolver

### Server Types Explained

| Server Type                       | Description                                                                                                                                                                                                                                                                                  |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **DNS Root Server**               | Responsible for answering queries about TLDs (for example, `.com`, `.org`). Root servers are queried when the resolver does not know where to go next. ICANN coordinates the DNS root system. There are 13 logical root server identities worldwide (each backed by many anycast instances). |
| **Authoritative Name Server**     | Holds the definitive records for a DNS zone (for example, `example.com`). Its answers are binding for that zone.                                                                                                                                                                             |
| **Non-authoritative Name Server** | Does not own a zone but can answer queries based on data learned from other servers (via recursion/iteration).                                                                                                                                                                               |
| **Caching DNS Server**            | Stores (caches) DNS answers for a period of time based on the record’s TTL to speed up future lookups.                                                                                                                                                                                       |
| **Forwarding Server**             | Forwards queries to another DNS server (common in corporate networks where internal DNS forwards external lookups to a central resolver).                                                                                                                                                    |
| **Resolver**                      | The component that performs name resolution for a client. This can be the OS stub resolver, a router, or a full recursive resolver (for example, public DNS services).                                                                                                                       |

---

## DNS Privacy and Encryption

Traditional DNS is largely unencrypted. This means DNS queries can often be observed by:

* Local network attackers (same Wi-Fi / LAN)
* ISPs
* Network monitoring appliances

To mitigate privacy and interception risks, modern environments increasingly use:

* **DNS over TLS (DoT)**
* **DNS over HTTPS (DoH)**
* **DNSCrypt**

These protect the query transport between the client and resolver, but they do not automatically prevent all metadata leakage (for example, SNI, IP-level traffic patterns, or resolver-side logging).

---

## DNS Records

DNS is not only “name to IP”. It also stores service and infrastructure metadata that is extremely useful during reconnaissance.

| Record    | Description                                                                                         |
| --------- | --------------------------------------------------------------------------------------------------- |
| **A**     | Maps a name to an IPv4 address.                                                                     |
| **AAAA**  | Maps a name to an IPv6 address.                                                                     |
| **MX**    | Defines mail exchangers for a domain (mail servers).                                                |
| **NS**    | Defines authoritative name servers for the zone.                                                    |
| **TXT**   | Free-form text. Commonly used for verification tokens and email security (SPF, DKIM, DMARC).        |
| **CNAME** | Alias from one name to another canonical name.                                                      |
| **PTR**   | Reverse lookup record (IP → hostname) in reverse DNS zones.                                         |
| **SOA**   | Start of Authority record: defines zone-level metadata (primary NS, admin contact, serial, timers). |

---

## SOA Record (Start of Authority)

The SOA record describes who “owns” and manages the DNS zone and includes metadata required for zone replication and caching behaviour.

Example query:

```bash
dig soa www.inlanefreight.com
```

Example output:

```text
inlanefreight.com.      900     IN      SOA     ns-161.awsdns-20.com. awsdns-hostmaster.amazon.com. 1 7200 900 1209600 86400
```

In SOA records, the admin contact is stored in a hostname-like format where the first dot is interpreted as `@`. For example:

* `awsdns-hostmaster.amazon.com.` → `awsdns-hostmaster@amazon.com`

---

# Default Configuration (BIND9)

On Linux environments, **BIND9** is a very common DNS server implementation. DNS servers typically rely on:

* Local DNS configuration files
* Zone files (forward resolution)
* Reverse lookup zone files (reverse resolution)

BIND’s main configuration (`named.conf`) is commonly split across multiple files such as:

* `named.conf.local`
* `named.conf.options`
* `named.conf.log`

A key concept is the difference between **global options** and **zone options**:

* Global options apply to the entire server.
* Zone options apply only to a specific zone.
* If an option exists in both, the zone option usually takes precedence.

---

## Local DNS Configuration

Example `named.conf.local`:

```bash
cat /etc/bind/named.conf.local
```

Example contents:

```text
//
// Do any local configuration here
//

// Consider adding the 1918 zones here, if they are not used in your
// organization
//include "/etc/bind/zones.rfc1918";
zone "domain.com" {
    type master;
    file "/etc/bind/db.domain.com";
    allow-update { key rndc-key; };
};
```

This defines a zone and points BIND at the zone file where records are stored.

---

## Zone Files (Forward Resolution)

A zone file describes all records for a DNS zone using the BIND zone file format. A valid zone file must contain:

* Exactly **one SOA** record
* At least **one NS** record

If the zone file has syntax errors, BIND may treat it as unusable, and queries can result in `SERVFAIL`, effectively behaving as if the zone does not exist.

Example zone file:

```bash
cat /etc/bind/db.domain.com
```

Example contents:

```text
$ORIGIN domain.com
$TTL 86400
@     IN     SOA    dns1.domain.com.     hostmaster.domain.com. (
                    2001062501 ; serial
                    21600      ; refresh after 6 hours
                    3600       ; retry after 1 hour
                    604800     ; expire after 1 week
                    86400 )    ; minimum TTL of 1 day

      IN     NS     ns1.domain.com.
      IN     NS     ns2.domain.com.

      IN     MX     10     mx.domain.com.
      IN     MX     20     mx2.domain.com.

             IN     A       10.129.14.5

server1      IN     A       10.129.14.5
server2      IN     A       10.129.14.7
ns1          IN     A       10.129.14.2
ns2          IN     A       10.129.14.3

ftp          IN     CNAME   server1
mx           IN     CNAME   server1
mx2          IN     CNAME   server2
www          IN     CNAME   server2
```

This is effectively the “phone book” for forward lookups: it maps names to infrastructure roles and IPs.

---

## Reverse Name Resolution Zone Files

Reverse DNS maps **IP addresses → hostnames** using PTR records. These live in special reverse zones like:

* `in-addr.arpa` for IPv4
* `ip6.arpa` for IPv6

Example reverse zone file:

```bash
cat /etc/bind/db.10.129.14
```

Example contents:

```text
$ORIGIN 14.129.10.in-addr.arpa
$TTL 86400
@     IN     SOA    dns1.domain.com.     hostmaster.domain.com. (
                    2001062501 ; serial
                    21600      ; refresh after 6 hours
                    3600       ; retry after 1 hour
                    604800     ; expire after 1 week
                    86400 )    ; minimum TTL of 1 day

      IN     NS     ns1.domain.com.
      IN     NS     ns2.domain.com.

5    IN     PTR    server1.domain.com.
7    IN     MX     mx.domain.com.
...SNIP...
```

Reverse zones are frequently neglected in internal environments, but they can leak valuable naming conventions and internal host roles when properly configured.

---

# Dangerous Settings

DNS is highly sensitive operationally. Administrators often prioritise availability and “it works” behaviour, which can lead to permissive configurations.

Some BIND options commonly associated with dangerous exposure:

| Option            | Description                                                                           |
| ----------------- | ------------------------------------------------------------------------------------- |
| `allow-query`     | Which hosts are allowed to query the server.                                          |
| `allow-recursion` | Which hosts are allowed to use the server as a recursive resolver.                    |
| `allow-transfer`  | Which hosts are allowed to perform zone transfers (AXFR/IXFR).                        |
| `zone-statistics` | Enables statistics collection (can increase information exposure depending on setup). |

From an attacker perspective, the most impactful misconfiguration is often:

* **Zone transfers enabled to broad ranges or to everyone**

---

# Footprinting the Service

DNS footprinting is based on the queries you send. Even without active scanning, DNS can reveal a lot about infrastructure, services, and internal naming conventions.

## Query Name Servers (NS)

You can query NS records from a specific DNS server using `@<server>`:

```bash
dig ns inlanefreight.htb @10.129.14.128
```

This can reveal:

* Authoritative NS hostnames
* Additional A/AAAA records in the “Additional Section”
* Potential alternate servers that may have different security controls

---

## Query BIND Version (CHAOS)

Sometimes administrators leave version queries enabled via the CHAOS class:

```bash
dig CH TXT version.bind @10.129.120.85
```

If enabled, this can reveal the exact BIND version string, which can help map known vulnerabilities and hardening expectations. Many hardened environments disable this.

---

## ANY Query (Limited Use)

An `ANY` query asks for “whatever you are willing to give me”:

```bash
dig any inlanefreight.htb @10.129.14.128
```

Important notes:

* Modern DNS servers often restrict or minimise `ANY` responses.
* It does not guarantee a full zone dump.
* It can still leak valuable records like TXT (SPF verification), SOA, and NS.

---

# Zone Transfers (AXFR)

A zone transfer (AXFR) is a bulk transfer of an entire zone file, typically over TCP/53, used for replication between primary (master) and secondary (slave) DNS servers.

In legitimate operations:

* Primary/master contains the authoritative source of the zone.
* Secondary/slave periodically checks the SOA serial number and requests updates when the master serial increases.

If `allow-transfer` is misconfigured, an attacker can request the entire zone and obtain:

* Full host inventory
* Subdomains and service names
* Internal IP ranges (in split DNS scenarios)
* Naming conventions (DCs, VPN, mail, WSUS, apps)

## AXFR Attempt

```bash
dig axfr inlanefreight.htb @10.129.14.128
```

Example output (truncated):

```text
inlanefreight.htb.      604800  IN      SOA     inlanefreight.htb. root.inlanefreight.htb. 2 604800 86400 2419200 604800
...
app.inlanefreight.htb.  604800  IN      A       10.129.18.15
internal.inlanefreight.htb. 604800 IN   A       10.129.1.6
mail1.inlanefreight.htb. 604800 IN      A       10.129.18.201
ns.inlanefreight.htb.   604800  IN      A       10.129.34.136
```

### AXFR Against Internal Zones

If an internal zone is exposed:

```bash
dig axfr internal.inlanefreight.htb @10.129.14.128
```

Example output (truncated):

```text
dc1.internal.inlanefreight.htb. 604800 IN A     10.129.34.16
dc2.internal.inlanefreight.htb. 604800 IN A     10.129.34.11
vpn.internal.inlanefreight.htb. 604800 IN A     10.129.1.6
wsus.internal.inlanefreight.htb. 604800 IN A    10.129.18.2
...
```

This type of exposure is often a direct path to rapid infrastructure mapping and follow-on attacks (SMB, LDAP, Kerberos, VPN gateways, mail services, patch management servers, etc.).

---

# Subdomain Brute Forcing

If zone transfer is not possible, you can still discover hosts via brute force enumeration. This requires:

* A wordlist of likely subdomains/hostnames
* A target domain
* A DNS server to query (authoritative if possible)

Example bash loop:

```bash
for sub in $(cat /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt); do
  dig $sub.inlanefreight.htb @10.129.14.128 \
  | grep -v ';\|SOA' \
  | sed -r '/^\s*$/d' \
  | grep $sub \
  | tee -a subdomains.txt
done
```

Example hits:

```text
ns.inlanefreight.htb.     604800  IN  A  10.129.34.136
mail1.inlanefreight.htb.  604800  IN  A  10.129.18.201
app.inlanefreight.htb.    604800  IN  A  10.129.18.15
```

---

## Automated Tooling (dnsenum)

Tools like `dnsenum` automate common steps such as:

* NS/MX discovery
* Zone transfer attempts
* Brute-force subdomain enumeration

Example usage:

```bash
dnsenum --dnsserver 10.129.14.128 --enum -p 0 -s 0 -o subdomains.txt \
-f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt inlanefreight.htb
```

Even with automation, manual verification is important. Different tools behave differently with timeouts, resolver settings, recursion, wildcard DNS behaviour, and how they parse results.
