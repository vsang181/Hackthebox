## Service Enumeration

Accurately identifying running services and their versions is a critical step during enumeration. Precise version information allows us to research known vulnerabilities, locate version-specific exploits, and understand how a service behaves on a given operating system. The more exact the version fingerprint, the higher the likelihood of finding a reliable and targeted attack vector.

---

## Service Version Detection

A practical approach is to begin with a **lightweight port scan** to identify exposed ports with minimal network noise. This reduces the risk of early detection by defensive controls such as firewalls, IDS, or IPS systems. Once interesting ports are identified, a **service version scan** can be performed against those ports to determine application names and versions.

A full port scan (`-p-`) combined with service detection (`-sV`) is comprehensive but time-consuming, so it is often run in parallel or after initial triage.

Example:

```
sudo nmap 10.129.2.28 -p- -sV
```

| Option        | Description                       |
| ------------- | --------------------------------- |
| `10.129.2.28` | Target host                       |
| `-p-`         | Scan all TCP ports                |
| `-sV`         | Perform service version detection |

---

## Monitoring Scan Progress

Long-running scans can be monitored in real time.

### Using the Space Bar

Pressing the **space bar** during a scan displays live progress information, including elapsed time, completion percentage, and estimated time remaining.

### Periodic Status Updates

Progress output can also be automated using the `--stats-every` option:

```
sudo nmap 10.129.2.28 -p- -sV --stats-every=5s
```

| Option             | Description                         |
| ------------------ | ----------------------------------- |
| `--stats-every=5s` | Display scan status every 5 seconds |

---

## Increasing Verbosity

Increasing verbosity allows Nmap to report discoveries as they happen, such as newly identified open ports.

```
sudo nmap 10.129.2.28 -p- -sV -v
```

| Option | Description                                         |
| ------ | --------------------------------------------------- |
| `-v`   | Increase verbosity (use `-vv` for even more detail) |

This is especially useful for large scans, as it provides immediate feedback without waiting for completion.

---

## Banner Grabbing and Service Identification

Once scanning completes, Nmap reports open ports along with detected services and versions:

* Port number and protocol
* Service name
* Application version (when identifiable)
* OS or host hints (when available)

Nmap primarily relies on **banner grabbing**—reading identification strings returned by services after a successful connection. If banners are insufficient or ambiguous, Nmap falls back to **signature-based probes**, which increases scan duration but improves accuracy.

---

## Limitations of Automated Detection

Automated service detection does not always capture all available information. Services may:

* Suppress or modify banners
* Delay responses
* Return data Nmap does not fully parse or display

In some cases, raw packet inspection reveals additional details not shown in the summarized output.

---

## Manual Banner Verification

Manually connecting to a service can expose information missed during automated scanning.

### Example: SMTP Banner Inspection

Using `nc` (Netcat) to connect directly:

```
nc -nv 10.129.2.28 25
```

Typical output:

```
220 inlane ESMTP Postfix (Ubuntu)
```

This reveals:

* Mail server type (Postfix)
* Host identifier
* Underlying operating system (Ubuntu)

---

## Inspecting Traffic with Tcpdump

Capturing traffic during manual interaction allows inspection of protocol behavior and server responses.

Example capture command:

```
sudo tcpdump -i eth0 host 10.10.14.2 and 10.129.2.28
```

From the packet capture, the interaction follows the standard TCP flow:

1. **SYN** – Client initiates connection
2. **SYN-ACK** – Server responds
3. **ACK** – Connection established
4. **PSH-ACK** – Server sends banner data
5. **ACK** – Client acknowledges receipt

The banner is transmitted in a packet marked with the **PSH** flag, indicating that application-layer data is being delivered immediately to the client.
