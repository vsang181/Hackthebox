# Proxies

The word *proxy* is used loosely, and depending on who you ask, it can mean very different things:

* **Security professionals** often think of HTTP interception tools (for example, Burp Suite) or tunnelling via SOCKS/SSH proxies.
* **Web developers** usually associate proxies with services like Cloudflare or ModSecurity that sit in front of applications.
* **Everyday users** may believe a proxy is something that hides their location to access geo-restricted content.
* **Law enforcement** may associate proxies with attempts to obscure activity.

Not all of these interpretations are accurate.

At its core, a **proxy** is a device or service that **sits in the middle of a connection and actively mediates traffic**. The key word here is *mediates*. A true proxy can **inspect, understand, and potentially modify** the contents of the traffic passing through it. If a device only forwards traffic without inspection, it is technically acting as a **gateway**, not a proxy 

If you ever struggle to remember this distinction, think in terms of the OSI model: **proxies almost always operate at Layer 7**.

---

## Common Proxy Types

There are many specialised proxy implementations, but the most important ones you should understand are:

* Forward Proxy (Dedicated Proxy)
* Reverse Proxy
* Transparent vs Non-Transparent Proxy

These concepts appear frequently in both defensive architectures and offensive techniques.

---

## Forward Proxy (Dedicated Proxy)

A **forward proxy** is what most people imagine when they hear the word *proxy*.

In this model:

* The **client** sends a request to the proxy
* The **proxy** makes the request on behalf of the client
* The response is returned through the proxy

The destination server never communicates directly with the original client.

### Real-World Example

In many corporate environments, internal systems are **not allowed direct Internet access**. Instead, all outbound web traffic must pass through a forward proxy or web filter.

This provides several security benefits:

* Centralised inspection of outbound traffic
* Malware command-and-control traffic is easier to detect
* Policy enforcement (categories, domains, file types)

From an attacker’s perspective, this creates friction. Malware must either:

* Be **proxy-aware**, or
* Use an alternative C2 channel (for example, DNS tunnelling)

On Windows systems, this matters because:

* Browsers like Edge and Chrome obey **system proxy settings**
* Malware using native WinSock APIs may automatically inherit proxy support
* Firefox uses its own networking stack, meaning malware would need additional logic to extract Firefox-specific proxy settings

This significantly reduces the likelihood of simple malware working in tightly controlled environments.

### Security Tool Example

**Burp Suite** is a classic example of a forward proxy used during testing. While it is commonly used to forward HTTP requests, it is extremely flexible and can also be configured as a reverse or transparent proxy.

<img width="2576" height="1496" alt="forward_proxy" src="https://github.com/user-attachments/assets/814fec6d-ed59-42e9-81c3-2c79bbd54521" />

---

## Reverse Proxy

A **reverse proxy** flips the direction of mediation.

Instead of filtering outbound requests from clients, it:

* Listens for **incoming connections**
* Forwards them to one or more internal servers

Clients believe they are communicating directly with the server, but in reality, all traffic passes through the reverse proxy first.

### Common Use Cases

* **DDoS protection** (for example, Cloudflare)
* Load balancing
* TLS termination
* Web Application Firewalls (WAFs)

Services like Cloudflare sit in front of web applications and:

* Absorb large traffic volumes
* Filter malicious requests
* Hide the origin server’s real IP address

This significantly increases resilience and security for public-facing services.

### Offensive Use Case

From a penetration testing perspective, reverse proxies are also extremely useful.

For example:

* An infected host listens on a local port
* Incoming connections are forwarded back to the attacker
* Traffic is tunnelled through an existing channel (such as SSH)

This can:

* Bypass restrictive firewalls
* Evade perimeter IDS monitoring
* Reduce suspicious outbound traffic patterns

A common defensive reverse proxy is **ModSecurity**, a WAF that inspects HTTP requests and blocks those matching malicious patterns. Cloudflare can also act as a WAF, although this typically requires the organisation to allow HTTPS decryption.

<img width="2576" height="1526" alt="reverse_proxy" src="https://github.com/user-attachments/assets/ff2cc6c6-d326-42ed-aeb2-dd2611c35faf" />

---


## Transparent vs Non-Transparent Proxies

Proxies can also be categorised by **whether the client is aware of them**.

---

### Transparent Proxy

With a **transparent proxy**:

* The client is unaware that a proxy exists
* Traffic is intercepted automatically
* No client-side configuration is required

From the client’s perspective, communication appears direct. From the network’s perspective, the proxy silently substitutes itself into the connection.

These are commonly used by:

* ISPs
* Enterprise networks
* Content filtering systems

---

### Non-Transparent Proxy

With a **non-transparent proxy**:

* The client must be explicitly configured to use it
* Applications are told to send traffic to the proxy first
* Without this configuration, communication fails

In many corporate environments, this is enforced by design. If the proxy settings are missing or incorrect, the system simply cannot reach the Internet.

---

## Why Proxies Matter in Security

Proxies sit at powerful choke points in a network.

They can:

* Enforce policy
* Inspect content
* Block attacks
* Log activity
* Enable or restrict access paths

At the same time, they can also be:

* Abused for tunnelling
* Used to evade detection
* Leveraged for lateral movement

As you continue learning, always ask yourself:

**Who controls the proxy, and who benefits from it?**

Understanding that balance is critical for both attack and defence.
