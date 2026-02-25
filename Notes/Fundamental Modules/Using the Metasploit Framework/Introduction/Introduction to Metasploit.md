# Introduction to Metasploit

The **[Metasploit Framework](https://github.com/rapid7/metasploit-framework)** is a Ruby-based, modular penetration testing platform that provides a complete environment for writing, testing, and executing exploit code against vulnerable systems. Its module library contains hundreds of tested, real-world exploit proof-of-concepts that have been modularised for immediate use, covering a wide range of target operating systems, services, and application versions. Metasploit does not attempt to cover every conceivable attack surface; instead, it focuses on providing reliable, well-maintained coverage of the most commonly encountered unpatched vulnerabilities, making it a highly effective tool for the bulk of professional engagement work.

## Metasploit Framework vs. Metasploit Pro

Metasploit is distributed in two editions. The community **Framework** edition, accessed through `msfconsole`, covers the full exploit, payload, and post-exploitation workflow and is the version used throughout this module. **[Metasploit Pro](https://www.rapid7.com/products/metasploit/download/editions/)** extends the Framework with enterprise-focused capabilities across three operational areas:

| Infiltrate | Collect Data | Remediate |
|---|---|---|
| Manual exploitation and bruteforce | Discovery and Nexpose scan integration | Task chains and session rerun |
| AV and IDS/IPS evasion | Meta-modules and Project Sonar integration | Credential reuse and session management |
| Proxy and VPN pivoting | Social engineering and phishing wizards | Vulnerability validation and reporting |
| Web application testing | Evidence collection and data export | Team collaboration and backup/restore |

Metasploit Pro also includes a web interface and quick-start wizards for operators who prefer a GUI workflow, alongside the standard console for command-line users.

## MSFconsole

`msfconsole` is the primary and most fully featured interface to the Metasploit Framework. It is the only supported path to most of the Framework's functionality and provides a stable, readline-capable console with full tab completion and command history. External tools, scanners, and payload generators can be called directly from within the console, allowing operators to manage the complete assessment workflow, including enumeration, exploitation, session handling, and post-exploitation, from a single interface. Sessions and jobs can be managed simultaneously in a manner analogous to browser tabs, enabling efficient switching between multiple active target connections during post-exploitation work.

## Framework Architecture

All base files for the Metasploit Framework are located under `/usr/share/metasploit-framework` on Parrot OS Security and Kali Linux. Understanding the directory layout simplifies the process of adding new modules, writing custom exploits, and troubleshooting unexpected behaviour:

| Directory | Contents |
|---|---|
| `data/` and `lib/` | Core functioning components of the msfconsole interface and supporting libraries |
| `documentation/` | Full technical documentation for the project |
| `modules/` | All exploit, auxiliary, payload, encoder, evasion, NOP, and post modules |
| `plugins/` | Optional Ruby plugins that extend msfconsole functionality, including Nessus, OpenVAS, Nexpose, and sqlmap integrations |
| `scripts/` | Meterpreter scripts, resource scripts, and shell utilities |
| `tools/` | Command-line utilities for recon, payload generation, password attacks, module development, and memory dumping |

### Modules Directory

```bash
ls /usr/share/metasploit-framework/modules

auxiliary  encoders  evasion  exploits  nops  payloads  post
```

Each subdirectory maps directly to a module type. Custom modules or newly downloaded modules from the Rapid7 GitHub repository should be placed in the corresponding subdirectory to be recognised by `msfconsole` on the next launch or after issuing `reload_all`.

### Plugins Directory

```bash
ls /usr/share/metasploit-framework/plugins/

aggregator.rb      ips_filter.rb  openvas.rb           sounds.rb
alias.rb           komand.rb      pcap_log.rb          sqlmap.rb
auto_add_route.rb  lab.rb         request.rb           thread.rb
beholder.rb        libnotify.rb   rssfeed.rb           token_adduser.rb
db_credcollect.rb  msfd.rb        sample.rb            token_hunter.rb
db_tracker.rb      msgrpc.rb      session_notifier.rb  wiki.rb
event_tester.rb    nessus.rb      session_tagger.rb    wmap.rb
ffautoregen.rb     nexpose.rb     socket_logger.rb
```

Plugins are loaded manually within msfconsole using the `load` command or automatically at startup by configuring the `~/.msf4/msfconsole.rc` resource script. Notable plugins include `nessus.rb` and `openvas.rb` for vulnerability scanner integration, `sqlmap.rb` for database attack automation, and `pcap_log.rb` for packet capture logging during assessments.

### Scripts Directory

```bash
ls /usr/share/metasploit-framework/scripts/

meterpreter  ps  resource  shell
```

The `meterpreter/` subdirectory contains post-exploitation automation scripts run directly from within a Meterpreter session. The `resource/` subdirectory holds resource scripts, which are plain-text files containing sequences of msfconsole commands that automate repetitive setup tasks such as configuring listeners, setting common options, or executing multi-stage attack chains.
