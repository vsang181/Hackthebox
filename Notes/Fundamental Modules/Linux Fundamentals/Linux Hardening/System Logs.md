# System Logs on Linux

System logs are one of the most valuable sources of information on a Linux system. They record **what the system is doing, who is doing it, and when it happens**. As you analyse systems—either defensively or during a penetration test—you will rely heavily on logs to understand behaviour, identify weaknesses, and spot anomalies 

When you read logs correctly, they tell a story. Your job is to learn how to follow that story and recognise when something does not belong.

---

## Why System Logs Matter

System logs help you:

* Monitor system health and stability
* Investigate crashes or misbehaving services
* Detect unauthorised access attempts
* Identify cleartext credentials or weak authentication
* Track application behaviour and misuse

From a penetration testing perspective, logs can reveal:

* Successful and failed login attempts
* Evidence of brute-force attacks
* Misconfigured services
* Sensitive file access
* Security controls reacting (or failing to react) to your actions

After performing testing activities, reviewing logs also helps you understand **how visible your actions were**, which is useful when refining techniques.

---

## Log Configuration and Security

Logs are only useful if they are:

* Properly configured
* Stored securely
* Actively reviewed

You should ensure:

* Appropriate log levels are enabled
* Log rotation is configured to prevent disk exhaustion
* Logs are readable only by authorised users

Logs themselves are sensitive assets. If an attacker can modify or delete them, detection becomes much harder.

---

## Common Types of Linux Logs

Linux systems generate several categories of logs. You should know what each one contains and where to find it.

---

## Kernel Logs

**Kernel logs** record events related to the Linux kernel, including:

* Hardware detection
* Driver activity
* System calls
* Kernel errors and crashes

They are typically stored in:

```text
/var/log/kern.log
```

From a security point of view, kernel logs can expose:

* Vulnerable or outdated drivers
* Suspicious system calls
* Hardware-related instability that could lead to denial-of-service

Monitoring kernel logs helps you detect abnormal low-level behaviour that might otherwise go unnoticed.

---

## System Logs

**System logs** capture general system-level activity, such as:

* Service starts and stops
* Login attempts
* Scheduled jobs (cron)
* Reboots and shutdowns

They are commonly stored in:

```text
/var/log/syslog
```

### Example Syslog Entries

```text
Feb 28 2023 15:00:01 server CRON[2715]: (root) CMD (/usr/local/bin/backup.sh)
Feb 28 2023 15:04:22 server sshd[3010]: Failed password for htb-student from 10.14.15.2 port 50223 ssh2
Feb 28 2023 15:06:43 server apache2[2904]: 127.0.0.1 - - [28/Feb/2023:15:06:43 +0000] "GET /index.html HTTP/1.1" 200 13484 "-"
Feb 28 2023 15:07:19 server sshd[3010]: Accepted password for htb-student from 10.14.15.2 port 50223 ssh2
```

From entries like these, you can reconstruct timelines, identify attack attempts, and see whether access was successful.

---

## Authentication Logs

**Authentication logs** focus specifically on login activity and privilege escalation. These are some of the most important logs during a security assessment.

On Ubuntu systems, they are stored in:

```text
/var/log/auth.log
```

These logs record:

* Successful and failed SSH logins
* `sudo` usage
* PAM authentication events

### Example Authentication Log Entries

```text
Feb 28 2023 18:15:01 sshd[5678]: Accepted publickey for admin from 10.14.15.2 port 43210 ssh2
Feb 28 2023 18:15:03 sudo: admin : USER=root ; COMMAND=/bin/bash
Feb 28 2023 18:15:12 kernel: firewall: unexpected traffic allowed on port 22
```

From this, you can tell:

* Which authentication method was used
* Whether the user has `sudo` privileges
* Whether firewall behaviour was unexpected

As a penetration tester, this file is one of your first stops when checking for compromise or misconfiguration.

---

## Application Logs

**Application logs** are specific to individual services and applications. They provide insight into how an application processes requests and handles errors.

Common locations include:

* Apache:

  ```text
  /var/log/apache2/access.log
  /var/log/apache2/error.log
  ```
* Nginx:

  ```text
  /var/log/nginx/access.log
  ```
* MySQL:

  ```text
  /var/log/mysql/mysql.log
  ```
* PostgreSQL:

  ```text
  /var/log/postgresql/
  ```

These logs are especially valuable when testing web applications, APIs, or databases.

### Example Access Log Entry

```text
2023-03-07T10:15:23+00:00 servername privileged.sh: htb-student accessed /root/hidden/api-keys.txt
```

This immediately tells you:

* Which user acted
* What script was used
* Which sensitive file was accessed

Logs like this can expose privilege escalation paths or insecure scripts.

---

## Audit and Access Logs

Audit and access logs track **security-relevant actions**, such as:

* File access
* Configuration changes
* Execution of privileged commands

These logs are critical for detecting:

* Unauthorised access
* Data exfiltration
* Abuse of elevated privileges

Systemd-based systems may store these logs in:

```text
/var/log/journal/
```

---

## Security Logs

Security-focused tools maintain their own logs.

Examples include:

* Fail2ban:

  ```text
  /var/log/fail2ban.log
  ```
* UFW firewall:

  ```text
  /var/log/ufw.log
  ```

These logs show:

* Blocked IP addresses
* Repeated authentication failures
* Firewall decisions

As a tester, these logs help you understand **what defensive controls are active** and how they respond to attacks.

---

## Analysing Logs Effectively

You will rarely read logs manually from top to bottom. Instead, you should use command-line tools to filter and extract what matters.

Common tools include:

* `tail` – view recent log entries
* `grep` – search for keywords or patterns
* `sed` – transform and filter output

Proper log analysis allows you to:

* Build timelines
* Correlate events across services
* Identify subtle indicators of compromise

---

## Final Notes

Logs are not just for administrators. They are one of the most powerful tools you have as a penetration tester.

Learn where logs live. Learn what “normal” looks like. Once you understand that, anything abnormal stands out quickly.

If you ignore logs, you are working blind.
