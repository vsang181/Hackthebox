# Proxying Tools

Web proxies are not limited to browser traffic. Routing CLI tools and thick client applications through Burp or ZAP gives you the same visibility and control over their requests that you have over browser traffic. The setup principle is identical regardless of the tool: point it at `http://127.0.0.1:8080` and your proxy catches everything.

Keep in mind that proxying adds latency, so only route tools through the proxy when you need to inspect or manipulate their traffic. For normal usage, run them directly.

***

## Proxychains

Proxychains is the most versatile option on Linux because it wraps any command-line tool without requiring that tool to have native proxy support. It works by intercepting network calls at the system level and redirecting them through a configured proxy.

### Configuration

Edit `/etc/proxychains.conf`, comment out the default SOCKS line, and add your proxy at the bottom:

```
#socks4         127.0.0.1 9050
http 127.0.0.1 8080
```

### Usage

Prepend `proxychains` (with `-q` for quiet mode to suppress connection noise) before any command:

```bash
proxychains -q curl http://SERVER_IP:PORT
```

The `-q` flag suppresses proxychains' own output so you only see the tool's actual results. Every request made by the tool will now appear in your proxy's HTTP history.

This works with virtually any tool: `curl`, `wget`, `sqlmap`, `nikto`, `nmap` (for script-based HTTP requests), custom Python scripts, and anything else that makes outbound HTTP connections.

***

## Metasploit

Metasploit has built-in proxy support through the `PROXIES` option, which is available on most HTTP-based modules. There is no need for proxychains when using MSF directly.

```bash
msfconsole

msf6 > use auxiliary/scanner/http/robots_txt
msf6 auxiliary(scanner/http/robots_txt) > set PROXIES HTTP:127.0.0.1:8080
msf6 auxiliary(scanner/http/robots_txt) > set RHOST SERVER_IP
msf6 auxiliary(scanner/http/robots_txt) > set RPORT PORT
msf6 auxiliary(scanner/http/robots_txt) > run
```

The format for the PROXIES value is `PROTOCOL:HOST:PORT`. After running the module, every HTTP request it makes appears in your proxy history, letting you examine exactly what the module sends, verify it is hitting the right endpoints, and debug unexpected behaviour.

This is particularly useful when:

- A Metasploit module is not behaving as expected and you want to see the raw requests
- You want to manually replay or modify a request that the module generated
- You are demonstrating to a client exactly what traffic a specific exploit generates
