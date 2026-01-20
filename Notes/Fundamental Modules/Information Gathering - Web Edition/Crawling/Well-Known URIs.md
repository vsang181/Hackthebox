## Well-Known URIs

The **`.well-known`** standard, defined in **RFC 8615**, introduces a standardised directory at the root of a domain used to expose well-defined metadata and configuration information. This directory is accessible via the `/.well-known/` path and provides a predictable location for clients, applications, and security tools to discover information about a website’s services, protocols, and security policies.

By standardising the location of this data, `.well-known` removes the need for guesswork. Clients can construct a known URL to retrieve configuration details automatically. For example, a security policy can be accessed at:

```
https://example.com/.well-known/security.txt
```

The **Internet Assigned Numbers Authority (IANA)** maintains an official registry of `.well-known` URI suffixes. Each entry defines a specific purpose and is governed by a formal specification.

### Common .well-known URIs

| URI Suffix             | Description                                                | Status      | Reference                  |
| ---------------------- | ---------------------------------------------------------- | ----------- | -------------------------- |
| `security.txt`         | Contact information for reporting security vulnerabilities | Permanent   | RFC 9116                   |
| `change-password`      | Standard location for a password change page               | Provisional | W3C Draft                  |
| `openid-configuration` | OpenID Connect provider metadata                           | Permanent   | OpenID Connect Discovery   |
| `assetlinks.json`      | Verifies ownership between apps and domains                | Permanent   | Google Digital Asset Links |
| `mta-sts.txt`          | SMTP MTA Strict Transport Security policy                  | Permanent   | RFC 8461                   |

This is only a subset of the full registry, but it highlights how `.well-known` URIs are used across authentication, security, and infrastructure contexts.

---

## Web Recon and `.well-known`

From a reconnaissance perspective, `.well-known` endpoints are extremely valuable because they often expose **high-quality, structured metadata** intended for automation rather than humans. These endpoints frequently reveal:

* Authentication and authorization flows
* Security contact details
* Email security policies
* Third-party integrations
* Cryptographic key material (public keys)

One of the most powerful `.well-known` endpoints for recon is **OpenID Connect Discovery**.

---

## OpenID Connect Discovery (`openid-configuration`)

The `openid-configuration` endpoint is part of the **OpenID Connect Discovery** specification, which builds on OAuth 2.0. Client applications use this endpoint to automatically retrieve identity provider configuration details.

The endpoint is located at:

```
https://example.com/.well-known/openid-configuration
```

It returns a JSON document describing the identity provider’s capabilities and endpoints.

### Example Response

```json
{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth2/authorize",
  "token_endpoint": "https://example.com/oauth2/token",
  "userinfo_endpoint": "https://example.com/oauth2/userinfo",
  "jwks_uri": "https://example.com/oauth2/jwks",
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email"]
}
```

---

## Recon Value of `openid-configuration`

The data exposed by this endpoint enables deep insight into an application’s authentication architecture:

### Endpoint Discovery

* **Authorization Endpoint** – Entry point for login and consent flows
* **Token Endpoint** – Location where access and refresh tokens are issued
* **Userinfo Endpoint** – API that returns user attributes

### Cryptographic Details

* **JWKS URI** – Exposes public signing keys used to validate JWTs
* **Signing Algorithms** – Reveals supported cryptographic algorithms

### Capability Mapping

* **Supported Scopes** – Indicates what user data can be requested
* **Response Types** – Helps identify which OAuth/OpenID flows are enabled

From an offensive security perspective, this information can be used to:

* Map authentication flows
* Identify weak or deprecated algorithms
* Test token handling, validation, and audience restrictions
* Assess misconfigurations in OAuth/OpenID implementations
