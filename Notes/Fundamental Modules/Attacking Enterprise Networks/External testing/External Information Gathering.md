## External Information Gathering

### Initial Nmap Scan

The first step is a quick [Nmap](https://nmap.org) scan against the target to identify open ports. All output is saved to the relevant project subdirectory.

```bash
sudo nmap --open -oA inlanefreight_ept_tcp_1k -iL scope
```

This top 1,000 port TCP scan returns 11 open ports on `10.129.203.101`:

| Port | Service |
|------|---------|
| 21 | FTP |
| 22 | SSH |
| 25 | SMTP |
| 53 | DNS |
| 80 | HTTP |
| 110 | POP3 |
| 111 | RPCbind |
| 143 | IMAP |
| 993 | IMAPS |
| 995 | POP3S |
| 8080 | HTTP Proxy |

The host is running a web server alongside FTP, SSH, email services (SMTP, POP3, IMAP), DNS, and at least two web-facing ports.

### Full Port Scan with Aggressive Enumeration

A second scan runs across all ports using the `-A` flag, which enables OS detection, version scanning, and [Nmap script scanning (NSE)](https://nmap.org/book/man-misc-options.html). This is more intrusive than `-sV` alone, so care must be taken to ensure scripts do not cause service disruptions.

```bash
sudo nmap --open -p- -A -oA inlanefreight_ept_tcp_all_svc -iL scope
```

Key findings from this scan:

- FTP running [vsftpd 3.0.3](https://security.appspot.com/vsftpd.html) with anonymous login allowed and a `flag.txt` file visible
- SSH running [OpenSSH 8.2p1](https://www.openssh.com/) on Ubuntu
- SMTP running [Postfix](https://www.postfix.org/) with `VRFY` command enabled
- HTTP on ports 80 and 8080 both running [Apache httpd 2.4.41](https://httpd.apache.org/)
- Port 8080 identified as a potentially open proxy
- Email services running [Dovecot](https://www.dovecot.org/) for POP3 and IMAP on both plain and SSL ports
- Host OS is Ubuntu Linux

### Extracting Service Information

Use the following one-liner with the [nmap-grep cheatsheet](https://github.com/leonjza/awesome-nmap-grep) to extract clean service names from the `.gnmap` output:

```bash
egrep -v "^#|Status: Up" inlanefreight_ept_tcp_all_svc.gnmap | cut -d ' ' -f4- | tr ',' '\n' | \
sed -e 's/^[ \t]*//' | awk -F '/' '{print $7}' | grep -v "^$" | sort | uniq -c \
| sort -k 1 -nr
```

Output:

```
2 Dovecot pop3d
2 Dovecot imapd (Ubuntu)
2 Apache httpd 2.4.41 ((Ubuntu))
1 vsftpd 3.0.3
1 Postfix smtpd
1 OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
1 2-4 (RPC #100000)
```

### DNS Zone Transfer

DNS is open on port 53, so a zone transfer is attempted against the primary domain `INLANEFREIGHT.LOCAL` using [dig](https://linux.die.net/man/1/dig):

```bash
dig axfr inlanefreight.local @10.129.203.101
```

The zone transfer succeeds and returns 9 subdomains:

- `blog.inlanefreight.local`
- `careers.inlanefreight.local`
- `dev.inlanefreight.local`
- `gitlab.inlanefreight.local`
- `ir.inlanefreight.local`
- `status.inlanefreight.local`
- `support.inlanefreight.local`
- `tracking.inlanefreight.local`
- `vpn.inlanefreight.local`

If a zone transfer had not been possible, alternatives include [DNSDumpster](https://dnsdumpster.com/) for passive enumeration or the active subdomain enumeration methods covered in the [Information Gathering - Web Edition](https://academy.hackthebox.com/module/144/section/1256) module.

### Virtual Host Fuzzing with ffuf

Even with the zone transfer results in hand, vhost fuzzing with [ffuf](https://github.com/ffuf/ffuf) is run to catch anything the zone transfer may have missed. The wordlist used is the [namelist.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/DNS/namelist.txt) from [SecLists](https://github.com/danielmiessler/SecLists), located at `/opt/useful/seclists/Discovery/DNS/namelist.txt`.

First, determine the response size for a non-existent vhost to use as a filter baseline:

```bash
curl -s -I http://10.129.203.101 -H "HOST: defnotvalid.inlanefreight.local" | grep "Content-Length:"
```

A non-existent vhost returns a `Content-Length` of `15157`. Use this value with the `-fs` flag in ffuf to filter out invalid responses:

```bash
ffuf -w namelist.txt:FUZZ -u http://10.129.203.101/ -H 'Host:FUZZ.inlanefreight.local' -fs 15157
```

The ffuf results match all subdomains from the zone transfer and reveal one additional vhost not present in the DNS zone transfer output.

### Adding Hosts to /etc/hosts

All discovered subdomains are added to `/etc/hosts` for further investigation:

```bash
sudo tee -a /etc/hosts > /dev/null <<EOT

## inlanefreight hosts
10.129.203.101 inlanefreight.local blog.inlanefreight.local careers.inlanefreight.local dev.inlanefreight.local gitlab.inlanefreight.local ir.inlanefreight.local status.inlanefreight.local support.inlanefreight.local tracking.inlanefreight.local vpn.inlanefreight.local
EOT
```

### Enumeration Summary

Open ports and services identified for further probing:

- FTP with anonymous access enabled and a readable file present
- SSH with known version for potential exploit research
- SMTP with `VRFY` enabled, which can allow username enumeration
- Two Apache HTTP instances on ports 80 and 8080
- A potentially open HTTP proxy on port 8080
- Full DNS zone transfer exposing all internal subdomains

The next step is to dig deeper into individual services and probe each discovered subdomain for misconfigurations and exploitable vulnerabilities.
