# Attacking DNS

The [Domain Name System (DNS)](https://www.cloudflare.com/learning/dns/what-is-dns/) underpins virtually every network application, translating human-readable hostnames into IP addresses. Because it is fundamental infrastructure rather than an optional service, DNS misconfigurations expose reconnaissance data and attack paths that affect the entire organisation rather than a single host.

## Enumeration

DNS runs on UDP/53 by default, falling back to TCP/53 for large responses and zone transfers. An initial Nmap scan identifies the DNS software and version:

```bash
nmap -p53 -Pn -sV -sC 10.10.110.213
```

The version banner -- for example `ISC BIND 9.11.3` -- is immediately useful for identifying known CVEs specific to that software version.

## DNS Zone Transfer

A DNS zone transfer replicates zone data from a primary DNS server to secondary servers using the AXFR protocol. The critical security issue is that AXFR requires no authentication by default. Any client that can reach the DNS server can request a full copy of the zone unless the server is explicitly configured to restrict transfers to trusted IP addresses.

A successful zone transfer hands an attacker a complete map of the organisation's DNS namespace -- hostnames, IP addresses, internal naming conventions, mail servers, and any subdomains pointing to third-party services:

```bash
# Request a full zone transfer using dig
dig AXFR @ns1.inlanefreight.htb inlanefreight.htb
```

The output includes every record type in the zone: A records, AAAA records, MX records, CNAME records, NS records, and SOA data. From an attacker's perspective this is one of the most efficient reconnaissance steps possible -- a single query can return the entire internal DNS structure.

[Fierce](https://github.com/mschwager/fierce) automates the process by first identifying all nameservers for a domain and then attempting a zone transfer against each:

```bash
fierce --domain zonetransfer.me
```

The prevention is straightforward: configure the DNS server's `allow-transfer` directive to permit AXFR only from known secondary nameserver IP addresses, or block it entirely at the firewall level.

## Domain and Subdomain Takeover

Subdomain takeover occurs when a CNAME record in an organisation's DNS points to an external resource that no longer exists or has expired. If an attacker claims that external resource, they gain control over the subdomain without any access to the original organisation's DNS server.

The common pattern is:

1. A subdomain such as `support.target.com` has a CNAME pointing to a third-party service -- for example an AWS S3 bucket, a GitHub Pages site, a Heroku app, or a Fastly endpoint
2. The third-party resource is deleted or the account expires, but the CNAME record in the target's DNS is never removed
3. The external domain becomes available for registration, or the cloud resource becomes claimable
4. An attacker registers the resource and now controls what `support.target.com` resolves to

The practical impact is significant: serving malicious content, phishing under a trusted domain, stealing session cookies, or intercepting email if the takeover affects an MX record.

### Subdomain Enumeration Tools

Three tools cover most subdomain discovery scenarios:

[Subfinder](https://github.com/projectdiscovery/subfinder) scrapes subdomains from passive sources including certificate transparency logs, DNS aggregators, and open APIs:

```bash
./subfinder -d inlanefreight.com -v
```

[Subbrute](https://github.com/TheRook/subbrute) performs pure DNS brute-forcing using a wordlist and supports custom resolvers, making it useful during internal assessments where there is no internet access:

```bash
echo "ns1.inlanefreight.com" > ./resolvers.txt
./subbrute.py inlanefreight.com -s ./names.txt -r ./resolvers.txt
```

Once subdomains are discovered, check each CNAME record for dangling references:

```bash
host support.inlanefreight.com
```

A result like `support.inlanefreight.com is an alias for inlanefreight.s3.amazonaws.com` followed by a `NoSuchBucket` error when browsing to the URL confirms the subdomain is potentially takeable. The [can-i-take-over-xyz](https://github.com/EdOverflow/can-i-take-over-xyz) repository documents known takeover fingerprints for dozens of cloud providers and helps confirm whether a specific error message indicates a claimable resource.

## DNS Spoofing and Cache Poisoning

DNS cache poisoning replaces legitimate DNS records in a resolver's cache with attacker-controlled values, redirecting traffic for a target domain to a fraudulent IP without modifying the actual DNS server. Two primary approaches exist:

- **Network-level MITM** -- intercepting DNS queries in transit and injecting forged responses before the legitimate server replies
- **Server exploitation** -- compromising the DNS server directly and modifying its records

### Local Cache Poisoning with Ettercap

On a local network, [Ettercap](https://www.ettercap-project.org/) performs ARP poisoning to position itself between the victim and the gateway, then uses its `dns_spoof` plugin to serve fake DNS responses. The configuration maps target domains to the attacker's IP:

```bash
# Edit the DNS spoof config
cat /etc/ettercap/etter.dns

inlanefreight.com      A   192.168.225.110
*.inlanefreight.com    A   192.168.225.110
```

The workflow in Ettercap is:

1. Navigate to **Hosts > Scan for Hosts** to populate the host list
2. Add the victim's IP to Target1 and the default gateway IP to Target2
3. Navigate to **Plugins > Manage Plugins** and activate `dns_spoof`

Once active, any DNS query from the victim for `inlanefreight.com` or any subdomain of it returns `192.168.225.110` instead of the legitimate address. Browsing to the domain in a browser or running a ping from the victim confirms the redirect:

```cmd
C:\> ping inlanefreight.com
Pinging inlanefreight.com [192.168.225.110]
```

[Bettercap](https://www.bettercap.org/) provides the same capability with a more modern interface and scripting support, and is generally preferred for newer assessments.

DNS attacks presented here focus on reconnaissance and redirection. More advanced DNS attacks -- including DNSSEC bypass techniques, DNS tunnelling for data exfiltration, and Active Directory-integrated DNS abuse -- are covered in later modules.
