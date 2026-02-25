# Introduction to MSFconsole

`msfconsole` is the primary interface to the Metasploit Framework and the only supported path to the full breadth of its functionality. It is pre-installed on Kali Linux and Parrot OS Security and can be launched directly from any terminal. The `-q` flag suppresses the ASCII art banner and launches the console directly to the prompt, which is useful in time-constrained situations or when logging session output:

```bash
msfconsole        # launches with banner
msfconsole -q     # launches silently
```

Once inside, the `help` command lists all available console commands with descriptions. Tab completion and full readline support are available throughout the console, making command construction and module selection faster in practice.

## Keeping the Framework Current

The legacy `msfupdate` command is no longer the recommended update method on OS-managed installations. On Kali Linux, Parrot OS, and other Debian-based distributions where Metasploit is managed by the system package manager, updates are applied through `apt`:

```bash
sudo apt update && sudo apt install metasploit-framework
```

This ensures not only that the module library receives the latest additions, but also that all Ruby dependencies and supporting libraries are updated consistently with the framework. Running this before beginning an assessment ensures that recently published exploit modules are available. [hackers-arise](https://hackers-arise.com/metasploit-basics-part-14-updating-the-msfconsole/)

## Enumeration Before Exploitation

Before selecting any exploit module, thorough enumeration of the target is essential. Version numbers are the key output of this phase: a service's version determines whether a known vulnerability applies, and skipping enumeration leads to wasted time testing modules against targets that are either patched or running a different version than expected. The enumeration process should determine:

- Which services are running and on which ports
- The exact version of each service
- The operating system type and version
- Any application-level banners or headers that reveal additional version information

Unpatched or outdated service versions running on publicly accessible systems are the primary entry points in most Metasploit-assisted engagements. [notes.nikolamilekic](https://notes.nikolamilekic.com/Readwise/Articles/Using+the+Metasploit+Framework+-+Introduction+to+MSFconsole)

## MSF Engagement Structure

The Metasploit Framework engagement workflow follows five sequential stages, each with distinct objectives and corresponding module categories: [notes.nikolamilekic](https://notes.nikolamilekic.com/Readwise/Articles/Using+the+Metasploit+Framework+-+Introduction+to+MSFconsole)

<img width="1380" height="2588" alt="S04_SS03" src="https://github.com/user-attachments/assets/7030ef07-07e3-4d99-936f-ea316e29c257" />

```
Enumeration  →  Preparation  →  Exploitation  →  Privilege Escalation  →  Post-Exploitation
```

| Stage | Objective | Primary MSF Module Types |
|---|---|---|
| **Enumeration** | Identify services, versions, and potential vulnerabilities | Auxiliary (scanners, fingerprinters) |
| **Preparation** | Select and configure the appropriate exploit and payload | Exploit modules, payload selection |
| **Exploitation** | Deliver the payload and establish an initial session | Exploit modules |
| **Privilege Escalation** | Elevate from the initial session context to a higher-privilege account | Local exploit modules, post modules |
| **Post-Exploitation** | Extract credentials, pivot to adjacent systems, maintain access | Post modules, Meterpreter scripts |

This structure maps directly to the module categories in the Framework's directory layout. Auxiliary modules serve the enumeration stage; exploit modules serve preparation and exploitation; post modules serve privilege escalation and post-exploitation. Understanding which stage you are at during an assessment determines which module category to search within, making the workflow substantially more efficient than browsing the full module list without context.

## Essential MSFconsole Commands

A core set of commands covers the majority of day-to-day interaction with `msfconsole`: [offsec](https://www.offsec.com/metasploit-unleashed/msfconsole-commands/)

| Command | Function |
|---|---|
| `search <keyword>` | Search the module library by name, CVE, platform, or type |
| `use <module>` | Load a module into the active context |
| `show options` | Display all configurable options for the active module |
| `set <option> <value>` | Assign a value to a module option |
| `run` / `exploit` | Execute the active module |
| `sessions -l` | List all active sessions |
| `sessions -i <id>` | Interact with a specific session |
| `sessions -u <id>` | Upgrade a shell session to a Meterpreter session |
| `jobs -l` | List all background jobs |
| `back` | Unload the current module and return to the root prompt |
| `help` | Display all available commands |

The `search` command supports filter syntax for more targeted results. Filtering by module type narrows results significantly in a library of over 2,000 exploit modules: [offsec](https://www.offsec.com/metasploit-unleashed/msfconsole-commands/)

```bash
search type:auxiliary name:smb
search type:exploit platform:windows ms17-010
search type:post name:hashdump
```
