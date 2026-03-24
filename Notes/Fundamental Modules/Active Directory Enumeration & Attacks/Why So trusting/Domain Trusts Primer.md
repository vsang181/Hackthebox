# Domain Trusts Primer

A domain trust creates an authentication link between two domains or forests, allowing users to access resources outside their home domain. Trusts are commonly introduced during company acquisitions, MSP partnerships, or geographic expansions, and they frequently introduce unintended attack paths when security is not evaluated before the trust is established. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Trust Types

| Trust Type | Scope | Transitive | Notes |
|-----------|-------|-----------|-------|
| Parent-child | Within same forest | Yes | Two-way by default; child trusts parent automatically |
| Cross-link | Between child domains | Yes | Speeds up authentication, skips parent |
| External | Separate forests, no forest trust | No | Uses SID filtering to restrict cross-domain auth |
| Tree-root | Forest root to new tree root | Yes | Created automatically when new tree is added |
| Forest | Between two forest root domains | Yes | Most permissive cross-org trust |
| ESAE | Bastion/admin forest | Varies | Used for privileged AD management isolation |

Transitivity is the key concept to understand here. In a transitive setup, if Domain A trusts Domain B and Domain B trusts Domain C, then Domain A implicitly trusts Domain C. A non-transitive trust is a strict bilateral relationship: only the two named domains are involved, and trust does not extend further down the chain. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Trust Direction

Trust direction determines who can access what:

- **One-way:** Users in the trusted domain access resources in the trusting domain. The trusting domain opens its door; the trusted domain does not.
- **Bidirectional:** Both domains grant mutual access. If `INLANEFREIGHT.LOCAL` and `FREIGHTLOGISTICS.LOCAL` have a bidirectional trust, users from either side can authenticate into the other.

A bidirectional trust does not mean equal privilege. It means authentication is permitted both ways. A domain user from a trusted domain may only have standard user rights in the trusting domain, unless explicitly granted more. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Enumeration

### Built-in PowerShell AD Module

```powershell
Import-Module activedirectory
Get-ADTrust -Filter *
```

Key fields to note in the output:

- `IntraForest: True` means child domain within the same forest
- `ForestTransitive: True` means a forest trust with an external domain
- `Direction: BiDirectional` means authentication flows both ways
- `SIDFilteringQuarantined: False` means SID filtering is not enabled, which is relevant for attack chains

### PowerView

```powershell
# List all trusts from current domain
Get-DomainTrust

# Map all trusts recursively across all reachable domains
Get-DomainTrustMapping
```

`Get-DomainTrustMapping` is more thorough because it queries trusts from each domain it discovers, not just the current one. This reveals the full picture, including trusts that are not visible from your starting position. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### Enumerate Users Across a Trust

Once you have identified a trusted domain, you can begin enumerating its objects directly:

```powershell
Get-DomainUser -Domain LOGISTICS.INLANEFREIGHT.LOCAL | select SamAccountName
```

This works because the bidirectional trust allows authentication to flow to the child domain, so LDAP queries against it are permitted with your current credentials. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### netdom (CMD)

```cmd
# List domain trusts
netdom query /domain:inlanefreight.local trust

# List domain controllers
netdom query /domain:inlanefreight.local dc

# List workstations and servers
netdom query /domain:inlanefreight.local workstation
```

`netdom` is a clean built-in option when you want to avoid loading PowerShell modules or are working in a restricted environment. It does not provide as much detail as PowerView but confirms trust existence and direction quickly. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### BloodHound

The pre-built query `Map Domain Trusts` in BloodHound visualizes the full trust mesh graphically. This is the fastest way to understand a complex multi-domain environment at a glance, particularly when multiple bidirectional trusts exist across different forests. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Why Trusts Matter for Attackers

The most impactful scenario is the "end-around" attack: you cannot find a foothold in the primary target domain, but a trusted domain has weaker security. A successful attack against the trusted domain can provide credentials or a foothold that translates directly into access in the principal domain, particularly if:

- A user in the trusted domain is a member of a privileged group in the principal domain
- Kerberoasting in the trusted domain yields a service account that has rights elsewhere
- Unconstrained delegation is enabled on a host in the trusted domain, allowing TGT harvesting across the trust

Bidirectional forest trusts with `SIDFilteringQuarantined: False` are especially dangerous, as they do not filter out SID history attributes during authentication, which is a prerequisite for SID history injection attacks across forest boundaries. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Large organizations are often unaware that some of their trust relationships even exist, particularly after acquisitions. This makes trust enumeration a high-value early step in any internal assessment, and findings around insecure or unnecessary trusts are worth including in client reports even if no direct exploitation path is found.
