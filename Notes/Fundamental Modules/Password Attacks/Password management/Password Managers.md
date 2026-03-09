# Password Managers

With the average person now managing around 100 passwords, the choice between reusing weak credentials and memorising hundreds of unique ones is a false dilemma — a [password manager](https://en.wikipedia.org/wiki/Password_manager) resolves it by acting as a secure, encrypted vault that handles storage, generation, and autofill automatically.

## How Password Managers Work

Most password managers are built on [Zero-Knowledge Encryption](https://blog.cubbit.io/blog-posts/what-is-zero-knowledge-encryption), which means the provider's servers never hold anything usable — only encrypted ciphertext. Encryption happens entirely on the user's device before any data is transmitted. The typical key derivation chain works as follows:

1. The master password is fed into a [key derivation function (KDF)](https://en.wikipedia.org/wiki/Key_derivation_function) such as PBKDF2-SHA256, Argon2, or bcrypt — with a random salt and a high iteration count (commonly 100,000 or more) to make brute-force attacks computationally impractical
2. The output of the KDF becomes the encryption key used to protect the vault, and a separate hash derived from the master password is sent to the server for authentication only — never the password itself
3. Vault contents are encrypted and decrypted locally — the server stores only unreadable ciphertext

This is why the master password is the single most important element of the entire system. If it is weak, an attacker who obtains a stolen vault can attempt offline brute-force against it at their own pace, as demonstrated by the 2022 LastPass breach. Technical whitepapers from [Bitwarden](https://bitwarden.com/images/resources/security-white-paper-download.pdf), [1Password](https://1passwordstatic.com/files/security/1password-white-paper.pdf), and [LastPass](https://assets.cdngetgo.com/da/ce/d211c1074dea84e06cad6f2c8b8e/lastpass-technical-whitepaper.pdf) each document their specific implementation in detail.

## Cloud vs Local Password Managers

The tradeoff between cloud and local storage comes down to convenience versus control:

| Factor | Cloud-based | Local |
|---|---|---|
| Sync across devices | Automatic, built-in | Manual or third-party sync |
| Access if device lost | Yes, via cloud login | Only if backup exists |
| Attack surface | Provider's servers and your device | Your device only |
| Responsibility | Shared with provider | Entirely on the user |
| Offline access | Limited or none | Always available |

As [Dashlane's analysis](https://blog.dashlane.com/password-storage-cloud-versus-local/) notes, local storage is not inherently more secure — it shifts the risk rather than eliminating it. A poorly secured local machine is a more realistic threat for most users than a well-audited cloud provider.

### Cloud password managers

- [1Password](https://1password.com/) — strong security model with an additional Secret Key separate from the master password
- [Bitwarden](https://bitwarden.com/) — open-source, independently audited, free tier available
- [Dashlane](https://www.dashlane.com/) — includes a built-in VPN and dark web monitoring
- [Keeper](https://www.keepersecurity.com/) — strong enterprise and MSP focus
- [LastPass](https://www.lastpass.com/) — widely used but significantly damaged by the 2022 breach
- [NordPass](https://nordpass.com/) — uses XChaCha20 encryption, developed by NordVPN's team
- [RoboForm](https://www.roboform.com/) — strong form-fill capabilities, long-standing product

### Local password managers

- [KeePass](https://keepass.info/) — open-source, no cloud component, highly extensible with plugins
- [KWalletManager](https://apps.kde.org/kwalletmanager5/) — integrated into the KDE desktop on Linux
- [Pleasant Password Server](https://pleasantpasswords.com/) — team-focused, built on top of KeePass
- [Password Safe](https://pwsafe.org/) — originally designed by Bruce Schneier, minimal and audited

## Alternatives to Passwords

Password managers significantly reduce the risk of credential reuse and weak passwords, but the underlying authentication mechanism remains a shared secret. Several methods eliminate that dependency entirely:

- [FIDO2](https://fidoalliance.org/fido2/) — uses public-key cryptography where the private key never leaves the hardware device. During login, the server sends a challenge, the device signs it with the private key, and the server verifies against the stored public key — no password is ever transmitted or stored. Physical keys like [YubiKey](https://www.yubico.com/) implement this standard, with a broader list available via [Microsoft's supported FIDO2 providers](https://docs.microsoft.com/en-us/azure/active-directory/authentication/concept-authentication-passwordless#fido2-security-key-providers)
- [Multi-factor Authentication (MFA)](https://en.wikipedia.org/wiki/Multi-factor_authentication) — adds a second factor on top of a password, significantly raising the cost of account compromise even when credentials are leaked
- [TOTP](https://en.wikipedia.org/wiki/Time-based_one-time_password) — generates a time-limited one-time code from a shared secret; more resistant to replay attacks than static passwords but still vulnerable to real-time phishing
- IP restrictions and device compliance — tools like [Microsoft Endpoint Manager](https://www.petervanderwoude.nl/post/tag/device-compliance/) and [Workspace ONE](https://www.loginconsultants.com/enabling-the-device-compliance-with-workspace-one-uem-authentication-policy-in-workspace-one-access) enforce that authentication can only succeed from known, healthy devices regardless of credential validity

## The Passwordless Direction

Several major identity providers — including [Microsoft](https://www.microsoft.com/en-us/security/business/identity-access-management/passwordless-authentication), [Okta](https://www.okta.com/passwordless-authentication/), [Auth0](https://auth0.com/passwordless), and [Ping Identity](https://www.pingidentity.com/en/resources/blog/posts/2021/what-does-passwordless-really-mean.html) — are actively moving toward removing passwords from the authentication equation altogether. Passwordless authentication replaces the knowledge factor (something you know) with either a possession factor (something you have, such as a hardware key or a device with a stored passkey) or an inherence factor (something you are, such as a biometric).

Passkeys — the consumer-facing implementation of FIDO2 — gained widespread platform support across Apple, Google, and Microsoft ecosystems through 2023 and 2024, with adoption continuing to accelerate as more services offer them as a primary login method rather than a secondary option. From a security perspective, passkeys are phishing-resistant by design: a site-specific key pair means a credential harvested from one phishing site cannot be replayed against another.

The right combination of tools depends on the threat model of the individual or organisation. Using a password manager for service accounts and FIDO2 hardware keys for critical administrative access covers the majority of real-world attack scenarios addressed throughout this module.
