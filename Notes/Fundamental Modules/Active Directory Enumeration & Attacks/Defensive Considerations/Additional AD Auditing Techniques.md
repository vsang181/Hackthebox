# Additional AD Auditing Techniques

These tools serve a dual purpose during assessments: they help attackers enumerate the environment more thoroughly, and they provide defenders and clients with clear, reportable evidence of misconfigurations that might otherwise be difficult to communicate to management. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## AD Explorer (Sysinternals)

AD Explorer is part of the Microsoft Sysinternals Suite and functions as an advanced AD viewer and editor. Its most operationally useful feature for assessments is the ability to create offline snapshots of the entire AD database. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Key capabilities:

- Browse all AD objects and attributes without opening dialog boxes
- View and edit permissions and schema information
- Execute and save complex searches for re-use
- Create point-in-time snapshots for offline analysis during the reporting phase
- Compare two snapshots to identify changes in objects, attributes, and security permissions between scans

To create a snapshot, log in with any valid domain user, then navigate to `File > Create Snapshot`. The snapshot file can be moved offline and explored like any other database, which is particularly useful when you no longer have access to the environment but need to review AD object properties during report writing. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## PingCastle

PingCastle takes a different approach from tools like BloodHound or PowerView. Rather than purely providing offensive enumeration data, it evaluates the domain's security posture using a risk assessment framework based on the Capability Maturity Model Integration (CMMI) and produces a scored report that is readable by both technical staff and management. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Run it interactively by launching the executable:

```cmd
PingCastle.exe
```

The TUI presents the following main options:

| Option | Purpose |
|--------|---------|
| `healthcheck` | Full domain risk score and misconfiguration overview |
| `conso` | Aggregate multiple domain reports into one |
| `carto` | Build a visual map of all interconnected/trusted domains |
| `scanner` | Run targeted security checks on workstations |
| `export` | Export user or computer lists |

The scanner submenu includes checks that directly map to attack techniques covered throughout this module:

```
1-aclcheck        Checks ACL permissions on AD objects
2-antivirus       Checks AV deployment status
5-laps_bitlocker  Checks LAPS and BitLocker deployment
6-localadmin      Enumerates local admin accounts
7-nullsession     Tests for null session access
e-spooler         Checks for exposed Print Spooler (Printer Bug)
g-zerologon       Tests Zerologon vulnerability
b-share           Enumerates accessible shares
c-smb             Checks SMB configuration
```

The healthcheck report produces an overall domain risk score and highlights anomalies requiring immediate attention, including trust relationships, delegation misconfigurations, share exposure, and recent vulnerability susceptibility. This scoring model makes it a strong tool for showing clients exactly where their domain falls on the security maturity scale and justifying remediation priorities. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## Group3r

Group3r is purpose-built to audit Group Policy Objects for vulnerabilities and misconfigurations. It runs from any domain-joined host using a standard domain user account, no admin rights required. If running from a non-domain-joined host, use `runas /netonly` to run in the context of a domain user. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Basic usage, outputting results to a log file:

```cmd
group3r.exe -f findings.log
```

Output flags:

- `-f <filepath>` writes results to a file
- `-s` sends results to stdout
- `-h` shows the full help menu

### Reading the Output

Group3r uses indentation levels to organise its findings:

- No indent: the GPO itself
- One indent: the policy setting within that GPO
- Further indent: the specific finding within that setting

Each finding is presented as a linked box showing the relevant policy setting, the interesting attribute, and the reason it was flagged. Group3r regularly surfaces issues that BloodHound and PowerView miss because it specifically focuses on GPO content rather than AD object relationships. Common findings include cleartext passwords in GPO scripts, weak permission assignments, and misconfigured software deployment settings. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

***

## ADRecon

ADRecon is the most comprehensive bulk collection tool of this group. It pulls a wide range of data in a single run and formats output into both CSV files and an HTML/Excel report. It is best used when stealth is not a concern and you want to ensure nothing was missed during manual enumeration. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

Run from a domain-joined host:

```powershell
.\ADRecon.ps1
```

Data collected in a single run includes:

- Domain, forest, trusts, sites, and subnets
- Schema history
- Default and fine-grained password policies
- Domain Controllers
- Users, SPNs, and password attributes
- Groups, membership, and membership changes
- OUs, GPOs, and scope of management links
- DNS zones and records
- Printers, computers, and SPNs
- LAPS and BitLocker recovery keys (requires privileged account)
- Full GPO report

Output is written to a timestamped folder in the execution directory:

```
ADRecon-Report-20220328092458\
    CSV-Files\          (raw data for each category)
    GPO-Report.html
    GPO-Report.xml
```

Two important requirements to be aware of before running: [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)

- Excel must be installed on the host for the tool to auto-generate the Excel report; without it, only CSVs are produced
- The `GroupPolicy` PowerShell module must be installed on the host to capture GPO output

If neither condition is met on your current host, collect the CSV files and generate the Excel report later on a suitable host using:

```powershell
.\ADRecon.ps1 -GenExcel C:\Tools\ADRecon-Report-20220328092458
```

***

## Choosing the Right Tool

| Tool | Best Use Case | Requires Admin | Output Format |
|------|--------------|---------------|---------------|
| AD Explorer | Offline snapshot, before/after comparison | No | Proprietary snapshot file |
| PingCastle | Client-facing risk scoring, executive reporting | No | HTML report, risk score |
| Group3r | GPO-specific vulnerability hunting | No | Log file / stdout |
| ADRecon | Comprehensive bulk collection, reporting | Partial (for LAPS/BitLocker) | CSV, HTML, Excel |

The overarching principle with all of these tools is that more evidence strengthens the report. A finding backed by PingCastle's risk score, a BloodHound attack path visualisation, and ADRecon CSV data is far more persuasive to a client's management team than a technical description alone, and gives their internal teams the concrete data they need to justify remediation funding. [ppl-ai-file-upload.s3.amazonaws](https://ppl-ai-file-upload.s3.amazonaws.com/web/direct-files/attachments/162556870/834886e2-c519-4551-bc23-22cc70ae5824/paste.txt)
