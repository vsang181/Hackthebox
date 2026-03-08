# Pass the Certificate

This section covers two distinct attack paths that both ultimately lead to obtaining a TGT using certificates rather than passwords or hashes. Here is a clean breakdown of what was covered.

## What Pass-the-Certificate Is

PKINIT is a Kerberos extension that allows certificate-based authentication instead of a password. Pass-the-Certificate abuses this by obtaining an X.509 certificate for a target account and using it to request a TGT directly from the KDC. The two attacks covered here are different ways to get that certificate.

## Attack Path 1: ESC8 (NTLM Relay to AD CS)

This exploits misconfigured Certificate Authority web enrollment over HTTP.

1. `ntlmrelayx` listens for inbound NTLM authentication and relays it to `/certsrv/certfnsh.asp`
2. The printer bug (`printerbug.py`) forces a domain controller machine account to authenticate back to your listener
3. The relay succeeds, and AD CS issues a certificate for `DC01$`
4. `gettgtpkinit.py` uses the `.pfx` certificate to obtain a TGT as `DC01$`
5. With a DC machine account TGT, `impacket-secretsdump` performs a DCSync to dump the Administrator NTLM hash

## Attack Path 2: Shadow Credentials (msDS-KeyCredentialLink)

This requires `WriteProperty` or `GenericWrite` over a target user, which BloodHound shows as the `AddKeyCredentialLink` edge.

1. `pywhisker` generates a certificate and writes its public key into the victim's `msDS-KeyCredentialLink` attribute
2. `gettgtpkinit.py` uses the generated `.pfx` to get a TGT as the victim user
3. You pass the ticket and authenticate as that user via Evil-WinRM or other tooling

## Attack Flow Comparison

| Step | ESC8 | Shadow Credentials |
|---|---|---|
| Prerequisite | CA with HTTP web enrollment | Write access to target's msDS-KeyCredentialLink |
| Coercion needed | Yes (printerbug or similar) | No |
| Certificate source | Relayed from DC via AD CS | Generated locally by pywhisker |
| Target | DC machine account | Any writable user/computer |
| End result | DCSync via DC$ TGT | Lateral movement as victim user |

## The No PKINIT Case

If the KDC does not support the required EKU for PKINIT (common in some environments), `PassTheCert` is the fallback. It authenticates directly over LDAPS using the certificate instead of going through Kerberos, and can be used to modify ACLs or change passwords to achieve the same end goal via a different path.
