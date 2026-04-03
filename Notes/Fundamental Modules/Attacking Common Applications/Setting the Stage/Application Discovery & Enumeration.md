## Application Discovery and Enumeration

Effective penetration testing starts with knowing what is running on the network. Many organisations lack a complete and accurate asset inventory, which means your enumeration work often has direct value to the client beyond just finding attack paths. You may uncover forgotten applications, expired demo software running without authentication, applications with default credentials, or instances running with known public vulnerabilities.

***

## Why Enumeration Matters

An organisation that does not know what is on its network cannot adequately protect it. As a tester you can deliver enumeration data to clients as formal findings, appendices listing identified services mapped to hosts, or supplemental scan data. Going further, educating clients on the tools used during this phase helps them perform proactive, periodic reconnaissance of their own networks before attackers do it for them.

***

## Notetaking and Organisation

Setting up a structured notetaking system before scanning begins saves significant time at the end of the engagement. Tools like [Obsidian](https://obsidian.md/), [Notion](https://www.notion.so/), [CherryTree](https://www.giuspen.net/cherrytree/), OneNote, or Evernote all work well. The choice comes down to personal preference, but the structure should be consistent across every engagement.

A recommended notebook layout for an external penetration test looks like this:

```
External Penetration Test - <Client Name>
├── Scope
│   └── In-scope IPs/ranges, URLs, fragile hosts, time constraints, limitations
├── Client Points of Contact
├── Credentials
├── Discovery/Enumeration
│   ├── Scans
│   ├── Live Hosts
│   └── Application Discovery
│       ├── Scans
│       └── Interesting/Notable Hosts
├── Exploitation
│   ├── <Hostname or IP>
│   └── <Hostname or IP>
└── Post-Exploitation
    ├── <Hostname or IP>
    └── <Hostname or IP>
```

Start the skeleton of the final report at the beginning of the assessment alongside your notebook. Fill in sections while scans are running to avoid a time crunch at the end.

***

## Initial Nmap Web Discovery

Start with a targeted port scan covering the most common web ports. Save the output in all formats using `-oA` so the XML file can be fed directly into screenshotting tools:

```bash
sudo nmap -p 80,443,8000,8080,8180,8888,10000 --open -oA web_discovery -iL scope_list
```

An example scope list file looks like this:

```
app.inlanefreight.local
dev.inlanefreight.local
drupal-dev.inlanefreight.local
drupal-qa.inlanefreight.local
drupal-acc.inlanefreight.local
drupal.inlanefreight.local
blog-dev.inlanefreight.local
blog.inlanefreight.local
app-dev.inlanefreight.local
jenkins-dev.inlanefreight.local
jenkins.inlanefreight.local
web01.inlanefreight.local
gitlab-dev.inlanefreight.local
gitlab.inlanefreight.local
support-dev.inlanefreight.local
support.inlanefreight.local
inlanefreight.local
10.129.201.50
```

Once the initial scan is done, run a deeper service version scan against hosts that return interesting results. The `-sV` flag pulls banner information and reveals running software versions:

```bash
sudo nmap --open -sV 10.129.201.50
```

Sample output identifying a Windows host running IIS, Splunk, and PRTG:

```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
135/tcp  open  msrpc         Microsoft Windows RPC
445/tcp  open  microsoft-ds?
3389/tcp open  ms-wbt-server Microsoft Terminal Services
8000/tcp open  http          Splunkd httpd
8080/tcp open  http          Indy httpd 17.3.33.2830 (Paessler PRTG bandwidth monitor)
8089/tcp open  ssl/http      Splunkd httpd (free license; remote login disabled)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Key things to note during this phase:

- Hosts with `dev` in the FQDN often run untested features or have debug mode enabled
- Windows hosts can be inferred from open ports like 135, 445, and 3389, though confirm this later
- [GitLab](https://about.gitlab.com/) instances are worth flagging early as they may allow open user registration and expose sensitive data in public repos or old commits

***

## Web Screenshotting with EyeWitness

[EyeWitness](https://github.com/FortyNorthSecurity/EyeWitness) takes screenshots of every web application found across all discovered ports and assembles them into an HTML report. It uses [Selenium](https://www.selenium.dev/) to render pages, fingerprints applications where possible, and suggests known default credentials for identified software. It accepts Nmap XML, Nessus XML, or a plain text file of URLs as input.

Install via apt or clone the repository and run the setup script:

```bash
sudo apt install eyewitness
```

```bash
git clone https://github.com/FortyNorthSecurity/EyeWitness.git
cd EyeWitness/Python/setup
./setup.sh
```

Run it against the Nmap XML output and save results to a named directory:

```bash
eyewitness --web -x web_discovery.xml -d inlanefreight_eyewitness
```

Key EyeWitness flags:

| Flag | Purpose |
|---|---|
| `--web` | Screenshot web targets using Selenium |
| `-x Filename.xml` | Nmap or Nessus XML input |
| `-f Filename` | Plain text URL list input |
| `--single URL` | Screenshot a single target |
| `-d Directory` | Output directory for the report |
| `--timeout` | Max seconds to wait per page (default: 7) |
| `--threads` | Number of concurrent threads |
| `--prepend-https` | Prepend http and https to bare URLs |

***

## Web Screenshotting with Aquatone

[Aquatone](https://github.com/shelld3v/aquatone) works similarly to EyeWitness and is actively maintained in the [shelld3v fork](https://github.com/shelld3v/aquatone). It accepts Nmap XML via the `-nmap` flag or Masscan XML, takes screenshots, clusters similar pages, and generates an HTML report.

Download and extract the binary:

```bash
wget https://github.com/michenriksen/aquatone/releases/download/v1.7.0/aquatone_linux_amd64_1.7.0.zip
unzip aquatone_linux_amd64_1.7.0.zip
```

Move it into your `$PATH` for convenient access from any directory:

```bash
sudo mv aquatone /usr/local/bin/
```

Run it against the Nmap XML output:

```bash
cat web_discovery.xml | ./aquatone -nmap
```

Aquatone completes quickly and writes its output to `aquatone_report.html` in the current directory. The report groups targets by response code and clusters visually similar pages together, making it faster to identify patterns across large scopes.

***

## Interpreting the Reports

Both tools organise results with high-value targets appearing first. Work through the entire report without skipping sections, since critical applications are sometimes buried deep in large outputs.

Priority items to flag immediately when reviewing screenshots:

- [Apache Tomcat](https://tomcat.apache.org/) manager interfaces at `/manager` and `/host-manager`, which allow WAR file uploads leading to RCE via [JSP](https://en.wikipedia.org/wiki/Jakarta_Server_Pages) if default credentials work
- [Jenkins](https://www.jenkins.io/) instances, particularly the [Script Console](https://www.jenkins.io/doc/book/managing/script-console/) which allows Groovy code execution
- [Splunk](https://www.splunk.com/) on port 8000, especially expired trial installations that may not require authentication
- [osTicket](https://osticket.com/) and other support ticketing systems that may expose sensitive customer data or allow domain email address registration
- [GitLab](https://about.gitlab.com/) instances allowing open self-registration, which may give access to private repos and historical commits containing credentials
- Printer admin pages, which sometimes expose cleartext LDAP credentials
- ESXi and vCenter portals, iLO and iDRAC pages, and network device login pages during internal assessments

During an external test, stay in the information gathering phase for the full discovery pass. Going down a rabbit hole on the first interesting host risks missing something more critical later in the report.

***

## Scan Discipline and Methodology

Scanners provide inputs to manual validation, not conclusions. The most severe and unique vulnerabilities are almost always found through manual testing rather than automated tools. Some practical rules to follow:

- Time and date stamp every scan
- Save all raw output files alongside the exact command syntax used
- Record the target scope for every scan run
- Keep all of this available in case the client questions specific activity seen in their logs
- Run [Nessus](https://www.tenable.com/products/nessus) alongside Nmap on non-evasive engagements to maximise coverage, but never rely on it exclusively

Since enumeration is iterative, run a screenshotting tool against every subsequent Nmap scan to ensure no web applications are missed on less common ports.
