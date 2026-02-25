# Sessions

`msfconsole` supports managing multiple simultaneous sessions and background jobs, allowing operators to interact with several compromised hosts in parallel, run additional post-exploitation modules against existing sessions, and keep exploit handlers running in the background without blocking the console prompt.

## Sessions

A session is created whenever an exploit or auxiliary module successfully establishes a communication channel with a target host. Sessions persist in the background while the operator works on other tasks, and the connection to the target remains active until either the session is explicitly terminated or the payload communication channel breaks down.

To background an active session, either press `[CTRL] + [Z]` or type `background` at a Meterpreter prompt. Both methods return the console to the `msf6 >` prompt while keeping the session alive:

```bash
meterpreter > background
[*] Backgrounding session 1...
msf6 exploit(windows/smb/psexec_psh) >
```

List all currently active sessions with the `sessions` command:

```bash
msf6 exploit(windows/smb/psexec_psh) > sessions

Active sessions
===============

  Id  Name  Type                     Information                 Connection
  --  ----  ----                     -----------                 ----------
  1         meterpreter x86/windows  NT AUTHORITY\SYSTEM @ MS01  10.10.10.129:443 -> 10.10.10.205:50501 (10.10.10.205)
```

Interact with a specific session using its index number:

```bash
msf6 exploit(windows/smb/psexec_psh) > sessions -i 1
[*] Starting interaction with 1...

meterpreter >
```

### Common Session Flags

| Flag | Function |
|---|---|
| `sessions` or `sessions -l` | List all active sessions |
| `sessions -i <id>` | Interact with a specific session |
| `sessions -k <id>` | Kill a specific session |
| `sessions -K` | Kill all active sessions |
| `sessions -u <id>` | Upgrade a standard shell session to Meterpreter |
| `sessions -s <script>` | Run a Meterpreter script on all active sessions |

## Running Post-Exploitation Modules Against Sessions

Backgrounding a session is most useful when chaining a second module against an already-compromised host. Load the desired post-exploitation module, then specify the session to run it against using the `SESSION` option in `show options`:

```bash
msf6 > use post/multi/recon/local_exploit_suggester
msf6 post(multi/recon/local_exploit_suggester) > set SESSION 1
SESSION => 1
msf6 post(multi/recon/local_exploit_suggester) > run
```

Post-exploitation modules fall into three broad categories commonly used against backgrounded sessions:

- **Credential gatherers:** Extract password hashes, plaintext credentials, and tokens from the target's memory and local stores.
- **Local exploit suggesters:** Enumerate the target's kernel version, installed patches, and running services to identify privilege escalation vectors.
- **Internal network scanners:** Pivot through the established session to enumerate hosts and services on internal network segments not directly reachable from the attacker's machine.

## Jobs

A job is a module running in the background within `msfconsole`. The key distinction between a session and a job is that a session represents an interactive connection to a target, while a job represents a module process running on the attacker's machine, such as an exploit handler listening on a port.

Terminating an active exploit handler with `[CTRL] + [C]` does not always release the bound port cleanly. Using the jobs interface to manage and terminate handlers avoids port conflicts when switching between listeners.

Run any exploit as a background job by appending `-j` to the `exploit` or `run` command:

```bash
msf6 exploit(multi/handler) > exploit -j
[*] Exploit running as background job 0.
[*] Started reverse TCP handler on 10.10.14.34:4444
```

List all running jobs:

```bash
msf6 exploit(multi/handler) > jobs -l

Jobs
====

 Id  Name                    Payload                    Payload opts
 --  ----                    -------                    ------------
 0   Exploit: multi/handler  generic/shell_reverse_tcp  tcp://10.10.14.34:4444
```

### Job Management Flags

| Flag | Function |
|---|---|
| `jobs -l` | List all running jobs |
| `jobs -i <id>` | Display detailed information about a specific job |
| `jobs -k <id>` | Terminate a job by its index number |
| `jobs -K` | Terminate all running jobs |
| `jobs -p <id>` | Add persistence to a job so it survives a console restart |
| `jobs -v` | Verbose output when combined with `-l` or `-i` |

## Session and Job Workflow

A typical workflow combining sessions and jobs in a multi-target engagement follows this pattern:

1. Launch the initial exploit handler as a background job with `exploit -j` so the listener persists without blocking the prompt.
2. Deliver the payload to the target through the appropriate vector. When the callback arrives, a new session is created automatically.
3. Background the new session with `[CTRL] + [Z]` and load the next required module.
4. Set the `SESSION` option to the relevant session index and run the module.
5. Use `sessions -l` and `jobs -l` to maintain a clear picture of all active connections and background processes throughout the assessment.
