# Databases

The Metasploit Framework integrates directly with [PostgreSQL](https://www.postgresql.org/) to provide persistent storage of scan results, discovered hosts, services, credentials, and loot across sessions. Without a connected database, all enumeration and exploitation results exist only in memory for the duration of the current console session. With one configured, findings are automatically recorded, searchable, and exportable, and can be used to populate module options such as `RHOSTS` directly from stored data rather than requiring manual entry.

## Database Setup

The setup process requires three sequential steps before `msfconsole` can connect:

```bash
# 1. Confirm PostgreSQL is running
sudo service postgresql status

# 2. Start PostgreSQL if it is not already active
sudo systemctl start postgresql

# 3. Initialise the MSF database schema and create the msf user
sudo msfdb init

[+] Creating database user 'msf'
[+] Creating databases 'msf'
[+] Creating databases 'msf_test'
[+] Creating configuration file '/usr/share/metasploit-framework/config/database.yml'
[+] Creating initial database schema
```

If `msfdb init` reports that the database is already configured, verify its status with `sudo msfdb status`. If `msfdb init` throws a `NoMethodError` or similar Ruby error, run `sudo apt update && sudo apt install metasploit-framework` first, then re-run initialisation.

Once the database is ready, launch `msfconsole` with the database connected in a single step using `sudo msfdb run`. To connect to an already-running instance from within an active console, verify the connection with `db_status`:

```bash
msf6 > db_status

[*] Connected to msf. Connection type: postgresql.
```

If the database needs to be fully reset, the reinitialisation sequence is:

```bash
msfdb reinit
cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/
sudo service postgresql restart
msfconsole -q
```

## Database Commands

The full set of database-related commands is accessible via `help database`:

| Command | Function |
|---|---|
| `db_status` | Confirm the current database connection type and status |
| `db_connect` | Connect to an existing external database instance |
| `db_disconnect` | Disconnect from the current database |
| `db_import` | Import scan results from an external file (auto-detects filetype) |
| `db_export` | Export the current workspace to a file in XML or pwdump format |
| `db_nmap` | Run Nmap and automatically store results in the database |
| `db_rebuild_cache` | Rebuild the database-stored module cache |
| `hosts` | List all stored host entries |
| `services` | List all stored service entries |
| `creds` | List and manage stored credentials |
| `loot` | List and manage stored loot items |
| `vulns` | List all stored vulnerabilities |
| `notes` | List all stored notes |
| `workspace` | Switch, create, or delete database workspaces |

## Workspaces

Workspaces function as project folders within the database. Each workspace maintains its own isolated set of hosts, services, credentials, loot, and notes, making it straightforward to separate findings from different targets, networks, or client engagements without entries from one assessment appearing alongside another.

```bash
msf6 > workspace
* default

msf6 > workspace -a Target_1
[*] Added workspace: Target_1
[*] Workspace: Target_1

msf6 > workspace Target_1
[*] Workspace: Target_1

msf6 > workspace
  default
* Target_1
```

The active workspace is marked with `*`. All subsequent scans, imports, and exploitation results are stored within the active workspace until it is changed. Workspace management options:

```bash
workspace                   # List all workspaces
workspace [name]            # Switch to a named workspace
workspace -a [name]         # Create a new workspace
workspace -d [name]         # Delete a workspace
workspace -D                # Delete all workspaces
workspace -r <old> <new>    # Rename a workspace
workspace -v                # List workspaces verbosely
```

## Importing and Running Scans

Nmap XML output is the preferred import format for `db_import`. XML preserves more structured data than plain text output and allows Metasploit to correctly parse host addresses, port states, service names, and version strings:

```bash
# Import a previously saved Nmap XML file
msf6 > db_import Target.xml

[*] Importing 'Nmap XML' data
[*] Importing host 10.10.10.40
[*] Successfully imported ~/Target.xml

# Verify the imported host and services
msf6 > hosts

address      mac  name  os_name  os_flavor  os_sp  purpose
-------      ---  ----  -------  ---------  -----  -------
10.10.10.40             Unknown                    device

msf6 > services

host         port   proto  name          state  info
----         ----   -----  ----          -----  ----
10.10.10.40  135    tcp    msrpc         open   Microsoft Windows RPC
10.10.10.40  445    tcp    microsoft-ds  open   Microsoft Windows 7 - 10 microsoft-ds
```

Nmap can also be executed directly from within `msfconsole` using `db_nmap`, which runs the scan and stores results simultaneously without requiring a separate terminal window or manual import step:

```bash
msf6 > db_nmap -sV -sS 10.10.10.8

[*] Nmap: PORT   STATE SERVICE VERSION
[*] Nmap: 80/tcp open  http    HttpFileServer httpd 2.3
[*] Nmap: Service Info: OS: Windows
```

The result is immediately queryable via `hosts` and `services` alongside any previously imported data.

## Populating RHOSTS from Database Results

One of the most practical advantages of the database integration is the ability to set `RHOSTS` directly from stored data rather than typing or pasting IP addresses manually. Both `hosts` and `services` support the `-R` flag for this:

```bash
# Set RHOSTS to all hosts currently stored in the workspace
msf6 > hosts -R

# Set RHOSTS to all hosts with port 445 open
msf6 > services -p 445 -R

# Set RHOSTS to hosts running a specific service by name
msf6 > services -s http -R
```

## Data Retention Commands

### hosts

The `hosts` table stores address, architecture, OS name, OS version, hostname, MAC address, and tag information. Entries are populated automatically from scans and can also be added or edited manually:

```bash
hosts -c address,os_name,purpose   # Display specific columns only
hosts -u                           # Show only hosts that are up
hosts -o report.csv                # Export to CSV
hosts -R                           # Set RHOSTS from the table
hosts -S <string>                  # Filter by search string
```

### services

The `services` table records open ports, protocol type, service name, and banner information per host. Filtering by port or protocol makes it efficient to identify all hosts running a specific service across a large workspace:

```bash
services -p 445,3389               # Filter by specific ports
services -s smb,http               # Filter by service name
services -r tcp                    # Filter by protocol
services -u                        # Show only services that are up
services -R                        # Set RHOSTS from results
```

### creds

The `creds` command stores and organises all credentials recovered during an assessment. It supports multiple credential types including plaintext passwords, NTLM hashes, SSH keys, and database-specific hash formats. Credentials can be added manually, filtered by type or port, and exported for use with offline cracking tools such as [Hashcat](https://hashcat.net/hashcat/) or [John the Ripper](https://www.openwall.com/john/):

```bash
creds                                      # List all stored credentials
creds -t ntlm                              # Filter by NTLM hashes only
creds -p 22,445                            # Filter by service port
creds add user:admin password:Password123  # Add a credential manually
creds -o hashes.jtr                        # Export in John the Ripper format
creds -o hashes.hcat                       # Export in Hashcat format
```

### loot

The `loot` table stores extracted files and data captured during post-exploitation, such as SAM database dumps, `/etc/shadow` extracts, and any other files retrieved from target systems. Items are stored with their host association, type, and an optional description:

```bash
loot                         # List all stored loot
loot -t hash                 # Filter by type
loot -S <string>             # Search by description
```

## Data Backup and Export

Export the current workspace before ending a session to ensure no data is lost in the event of a PostgreSQL service failure or system restart:

```bash
msf6 > db_export -f xml backup.xml

[*] Starting export of workspace default to backup.xml [ xml ]...
[*] Finished export of workspace default to backup.xml [ xml ]...
```

The export supports two formats: `xml` for full workspace backup including all hosts, services, credentials, loot, and notes; and `pwdump` for a credentials-only export in a format compatible with password cracking tools. The resulting XML file can be re-imported into any Metasploit instance using `db_import`.
