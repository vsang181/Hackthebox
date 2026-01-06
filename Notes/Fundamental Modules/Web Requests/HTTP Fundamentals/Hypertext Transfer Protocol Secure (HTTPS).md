# Hypertext Transfer Protocol Secure (HTTPS)

In the previous section, you looked at how **HTTP** requests are sent and processed. The major weakness of HTTP is simple but severe: **all data is transmitted in clear text**. This means anyone positioned between the client and the server can intercept and read the traffic using a **Man-in-the-Middle (MITM)** attack.

To solve this, **HTTPS (HTTP Secure)** was introduced. HTTPS encrypts all communication between the client and the server, making intercepted traffic unreadable to third parties. Because of this, HTTPS is now the **default standard on the modern web**, and plain HTTP is steadily being deprecated by browsers.

---

## Why HTTP Is Insecure

If you inspect an HTTP login request on the wire, you can clearly see sensitive data such as usernames and passwords transmitted in plain text.

<img width="2301" height="603" alt="https_clear" src="https://github.com/user-attachments/assets/9d0cfe48-f57d-45e5-b316-abd6b8541e69" />

Anyone on the same network (for example, public Wi-Fi) could capture this traffic and reuse the credentials. From a security perspective, this is unacceptable.

---

## What HTTPS Changes

When the same traffic is sent over HTTPS, the contents are **encrypted**. Even if the traffic is captured, it appears as unreadable encrypted data.

<img width="2203" height="598" alt="https_google_enc" src="https://github.com/user-attachments/assets/a70c85b4-2a14-4f40-ae16-3e32ffb015d6" />

At this point:

* Credentials are protected
* Session cookies are protected
* Request bodies and responses are protected

This is why HTTPS is considered mandatory for any modern web application.

---

## How to Identify HTTPS

You can easily recognise HTTPS usage in a browser by:

* The `https://` scheme in the URL
* A **lock icon** in the address bar

<img width="1692" height="667" alt="https_google" src="https://github.com/user-attachments/assets/e636f4fc-7c22-4662-b380-7cb754ff7401" />

If a site enforces HTTPS correctly, all communication between your browser and the server is encrypted.

---

## Important Caveat: DNS Is Still Visible

Even when HTTPS is used, **DNS lookups may still be unencrypted** if a clear-text DNS resolver is used. This can reveal which domains you are visiting.

To reduce this exposure, you can:

* Use encrypted DNS resolvers (for example, 8.8.8.8 or 1.1.1.1 with DoH/DoT)
* Use a VPN to encrypt all traffic end-to-end

---

## HTTPS Flow (High-Level)

<img width="2280" height="1960" alt="HTTPS_Flow" src="https://github.com/user-attachments/assets/2047f4de-d2ad-436a-9738-bedda1e781a4" />

At a high level, HTTPS works like this:

1. You request a site using `http://`
2. The server responds with a **301 redirect** to `https://`
3. Your browser connects to port **443**
4. A **TLS handshake** begins:

   * Client Hello
   * Server Hello
   * Certificate exchange
   * Key negotiation
5. An encrypted channel is established
6. Normal HTTP traffic continues **inside encryption**

Once the handshake completes, all data is transferred securely.

This explanation is intentionally simplified. The full cryptographic details are outside the scope of this module.

---

## Downgrade Attacks (Briefly)

In certain scenarios, attackers may attempt an **HTTP downgrade attack**, forcing a victim to use HTTP instead of HTTPS. This is usually done via a malicious MITM proxy.

Modern protections include:

* HSTS (HTTP Strict Transport Security)
* Browser warnings
* Secure cookie attributes

Because of these protections, downgrade attacks are far less effective today—but they still exist in poorly configured environments.

---

## Using cURL with HTTPS

`cURL` automatically handles HTTPS by:

* Performing the TLS handshake
* Verifying certificates
* Encrypting and decrypting traffic

If a site uses an **invalid or untrusted certificate**, cURL will refuse the connection by default:

```bash
curl https://inlanefreight.com
```

You may see an error similar to:

```
SSL certificate problem: Invalid certificate chain
```

This behaviour protects you against MITM attacks.

---

## Ignoring Certificate Validation (For Testing)

When testing:

* Local applications
* Labs
* Practice environments

You may encounter self-signed or invalid certificates. To bypass certificate verification **intentionally**, use the `-k` flag:

```bash
curl -k https://www.inlanefreight.com
```

This tells cURL to proceed **without validating the certificate**.

⚠️ You should only do this in controlled testing environments. Never ignore certificate errors on production systems you do not control.
