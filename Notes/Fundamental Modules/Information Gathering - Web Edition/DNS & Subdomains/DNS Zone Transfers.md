## DNS Zone Transfers

While brute-force enumeration is effective, **DNS zone transfers** represent a far more powerful (and quieter) technique when misconfigurations exist. A successful zone transfer can instantly reveal **every DNS record** for a domain, making it one of the most valuable findings during reconnaissance.

---

## What Is a DNS Zone Transfer?

A **DNS zone transfer** is a mechanism used to replicate DNS records between authoritative name servers. It ensures redundancy and consistency across primary and secondary DNS servers.

<img width="2613" height="1632" alt="ig_dns_zone_transfers_1" src="https://github.com/user-attachments/assets/e73f915c-47fa-4c4c-9764-934f6a8c96ed" />

In simple terms, it is a **bulk copy of the entire DNS zone file**, which includes:

* Subdomains
* IP addresses
* Mail servers
* Name servers
* Service records

### Typical Zone Transfer Flow

1. **AXFR Request**
   A secondary DNS server requests a full zone transfer using the `AXFR` query type.

2. **SOA Record Sent**
   The primary server responds with the **Start of Authority (SOA)** record, which includes the serial number and zone metadata.

3. **DNS Records Transmitted**
   All DNS records in the zone are transferred sequentially (A, AAAA, MX, NS, CNAME, TXT, SRV, PTR, etc.).

4. **Transfer Completion**
   The primary server signals that all records have been sent.

5. **Acknowledgement**
   The secondary server confirms successful receipt.

---

## Why Zone Transfers Are Dangerous

Zone transfers are **not inherently insecure**, but problems arise when **access controls are misconfigured**.

Historically, many DNS servers allowed **any client** to request a zone transfer. If this configuration persists, an attacker can download the entire zone file without authentication.

### Impact of an Unauthorised Zone Transfer

A successful zone transfer can expose:

* **All subdomains**
  Including internal, development, staging, admin, and legacy systems.

* **IP address mappings**
  Useful for pivoting into network-level attacks.

* **Mail and service infrastructure**
  MX, SRV, and TXT records may reveal internal services and security tooling.

* **Technology fingerprints**
  Records such as TXT, HINFO, or naming conventions can leak environment details.

This effectively provides a **complete DNS blueprint** of the target organisation.

---

## Remediation and Modern Practices

Most modern DNS servers restrict zone transfers to **explicitly trusted secondary name servers**. This is typically enforced using:

* IP-based allowlists
* TSIG (Transaction Signatures)
* Access Control Lists (ACLs)

Despite this, **misconfigurations still occur** due to:

* Legacy systems
* Human error
* Poor change management

For this reason, **testing for zone transfers (with authorisation)** remains a standard reconnaissance step.

---

## Attempting a Zone Transfer with `dig`

The `dig` tool can be used to request a full zone transfer.

### Syntax

```bash
dig axfr @<nameserver> <domain>
```

### Example

```bash
dig axfr @nsztm1.digi.ninja zonetransfer.me
```

If the server allows the transfer, the full DNS zone will be returned.

---

## Example Successful Zone Transfer Output (Excerpt)

```text
zonetransfer.me.      7200 IN SOA nsztm1.digi.ninja. robin.digi.ninja. ...
zonetransfer.me.      7200 IN MX 0 ASPMX.L.GOOGLE.COM.
zonetransfer.me.      7200 IN A 5.196.105.14
zonetransfer.me.      7200 IN NS nsztm1.digi.ninja.
zonetransfer.me.      7200 IN NS nsztm2.digi.ninja.
_acme-challenge.zonetransfer.me. IN TXT "..."
_sip._tcp.zonetransfer.me. IN SRV 0 0 5060 www.zonetransfer.me.
canberra-office.zonetransfer.me. IN A 202.14.81.230
```

This output demonstrates how **every record in the zone** becomes visible in a single request.

---

## Important Notes

* `zonetransfer.me` is **intentionally misconfigured** for educational purposes.
* Most real-world targets will **reject AXFR requests**.
* Even a failed attempt can still reveal:

  * Which name servers are authoritative
  * Whether TCP/53 is open
  * Error responses that hint at DNS software and configuration
