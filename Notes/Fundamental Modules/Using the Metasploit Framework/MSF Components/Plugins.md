# Plugins

Plugins are approved third-party integrations that extend `msfconsole` by loading additional commands, automation routines, and external tool bridges directly into the running console session. Unlike modules, which are self-contained scripts that perform a specific exploit or auxiliary task, plugins interact with the Metasploit API at a deeper level and can modify the console environment, add entirely new command groups, or establish live connections to external platforms such as vulnerability scanners. All findings surfaced through a plugin are automatically recorded to the active database workspace, removing the need to manually import results between tools.

## Loading Plugins

All pre-installed plugins are stored in `/usr/share/metasploit-framework/plugins/` as `.rb` files. Loading a plugin requires only the `load` command followed by the plugin name, without the `.rb` extension:

```bash
msf6 > load nessus

[*] Nessus Bridge for Metasploit
[*] Type nessus_help for a command listing
[*] Successfully loaded plugin: Nessus
```

Once loaded, the plugin's commands are immediately available from the `msfconsole` prompt. Run the plugin's help command to see the full list of available interactions:

```bash
msf6 > nessus_help

Command                     Help Text
-------                     ---------
nessus_connect              Connect to a Nessus server
nessus_login                Login with a different username
nessus_policy_list          List all policies
nessus_policy_del           Delete a policy
nessus_user_del             Delete a Nessus user
nessus_user_passwd          Change a Nessus user's password
<SNIP>
```

If a plugin is not installed or the filename does not match, `msfconsole` returns a clear error:

```bash
msf6 > load Plugin_That_Does_Not_Exist

[-] Failed to load plugin from /usr/share/metasploit-framework/plugins/Plugin_That_Does_Not_Exist.rb: 
cannot load such file -- /usr/share/metasploit-framework/plugins/Plugin_That_Does_Not_Exist.rb
```

## Installing Custom Plugins

Plugins not included in the default installation can be added by copying their `.rb` file into `/usr/share/metasploit-framework/plugins/`. [DarkOperator's Metasploit-Plugins](https://github.com/darkoperator/Metasploit-Plugins) repository is a well-known collection that provides several useful additions, including the `pentest.rb` plugin which adds automated discovery, credential collection, and post-exploitation orchestration commands:

```bash
# Clone the repository
git clone https://github.com/darkoperator/Metasploit-Plugins

# Copy the desired plugin to the plugins directory
sudo cp ./Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/pentest.rb
```

After copying, launch or reload `msfconsole` and load the plugin. A successful load extends the `help` output with the plugin's command groups:

```bash
msf6 > load pentest

      ___         _          _     ___ _           _
     | _ \___ _ _| |_ ___ __| |_  | _ \ |_  _ __ _(_)_ _
     |  _/ -_) ' \  _/ -_|_-<  _| |  _/ | || / _` | | ' \
     |_| \___|_||_\__\___/__/\__| |_| |_|\_,_\__, |_|_||_|
                                             |___/
Version 1.6
Pentest Plugin loaded.
by Carlos Perez (carlos_perez[at]darkoperator.com)
[*] Successfully loaded plugin: pentest


msf6 > help

Tradecraft Commands
===================
    check_footprint     Checks the possible footprint of a post module on a target system.

auto_exploit Commands
=====================
    show_client_side    Show matched client-side exploits from data imported from vuln scanners.
    vuln_exploit        Runs exploits based on data imported from vuln scanners.

Discovery Commands
==================
    discover_db             Run discovery modules against current hosts in the database.
    network_discover        Performs a port-scan and enumeration of services found for non-pivot networks.
    pivot_network_discover  Performs enumeration of networks available to a specified Meterpreter session.
    show_session_networks   Enumerate the networks one could pivot through Meterpreter in the active sessions.

Postauto Commands
=================
    app_creds           Run application password collection modules against specified sessions.
    multi_cmd           Run shell command against several sessions.
    multi_post          Run a post module against specified sessions.
    sys_creds           Run system password collection modules against specified sessions.
```

New plugins pushed by distribution maintainers are included automatically with standard Parrot OS and Kali Linux package updates via `apt`. Custom plugins sourced from external repositories must be copied manually each time.

## Popular Plugins

| Plugin | Status | Purpose |
|---|---|---|
| [Nmap](https://nmap.org/) | Pre-installed | Network discovery and port scanning integration |
| [Nessus](https://www.tenable.com/products/nessus) | Pre-installed | Tenable Nessus vulnerability scanner bridge |
| [NexPose](https://docs.rapid7.com/metasploit/vulnerability-scanning-with-nexpose/) | Pre-installed | Rapid7 Nexpose vulnerability management integration |
| [OpenVAS](https://www.openvas.org/) | Pre-installed | Open-source vulnerability scanner bridge |
| [Mimikatz v1](http://blog.gentilkiwi.com/mimikatz) | Pre-installed | Credential extraction from Windows memory |
| [Stdapi](https://www.rubydoc.info/github/rapid7/metasploit-framework/Rex/Post/Meterpreter/Extensions/Stdapi/Stdapi) | Pre-installed | Standard API for Meterpreter file system, network, and system commands |
| [Incognito](https://www.offensive-security.com/metasploit-unleashed/fun-incognito/) | Pre-installed | Windows token impersonation and privilege escalation |
| [Railgun](https://github.com/rapid7/metasploit-framework/wiki/How-to-use-Railgun-for-Windows-post-exploitation) | Pre-installed | Direct Windows API calls from within a Meterpreter session |
| [Darkoperator's collection](https://github.com/darkoperator/Metasploit-Plugins) | Manual install | Extended automation including pentest orchestration |

## Mixins

The Metasploit Framework is written in Ruby, a single-inheritance object-oriented language. Mixins are Ruby modules that provide a mechanism for including shared functionality across multiple classes without requiring a formal parent-child inheritance relationship. In practice, this means a Metasploit module can include the `Exploit::Remote::Tcp` mixin to gain all TCP connection handling methods without needing to inherit from a TCP class directly.

Mixins are relevant in two scenarios:

- **Adding optional features to a class:** A module that may or may not require authentication can include the relevant authentication mixin only when needed, keeping the core module clean.
- **Sharing one feature across many classes:** Protocol parsers, connection handlers, and encoding routines are implemented as mixins so that any module requiring that functionality can include it with a single `include` statement rather than reimplementing it.

Most of the Metasploit Framework's reusable capability, including TCP/UDP handling, HTTP client functionality, SMB interaction, and brute-force logic, is implemented as mixins located under `lib/msf/core/exploit/` and `lib/rex/`. Module developers include them by name at the top of their module definition. For further reading on mixin development, the [Metasploit UsingMixins guide](https://en.wikibooks.org/wiki/Metasploit/UsingMixins) covers the available mixin categories and how to apply them. For operators rather than developers, mixins have no direct impact on day-to-day module usage, but understanding that they exist explains much of the consistency in options and behaviour seen across modules that target different services.
