# The Concept of Attacks

Understanding how attacks work across different services becomes much simpler when there is a consistent framework to apply. Rather than learning each attack in isolation, the Concept of Attacks provides a four-category pattern -- Source, Process, Privileges, and Destination -- that maps onto virtually every vulnerability regardless of the protocol or service involved.

## The Four Categories

Every attack, regardless of complexity, follows the same underlying cycle. Information originates from a Source, gets handled by a Process, executes under a defined set of Privileges, and produces an outcome at a Destination. The value of this model is that it forces a structured analysis of why a vulnerability exists rather than just what it does.

<img width="1189" height="646" alt="image" src="https://github.com/user-attachments/assets/e4397731-afba-423e-b32b-dd52bab6589e" />

### Source

The Source is where the attacker-controlled input originates. Common input sources include:

- **Code** -- results from already-executed program logic used as input to further functions
- **Libraries** -- external collections of prebuilt code, configuration data, and subroutines integrated into an application
- **Config** -- static or prescribed values that shape how a process handles information
- **APIs** -- interfaces used to retrieve or provide information between components
- **User Input** -- direct manual entry of values that influence how information is processed

The Source is where the attacker gains influence over the system. It does not matter whether the protocol is HTTP, SMB, or FTP -- if an attacker can control what is fed into a process, the Source is the entry point.

### Process

The Process describes how the Source input is handled. This includes the program's data processing functions, the variables used as placeholders, and any logging behaviour. Most vulnerabilities exist here because developers cannot anticipate every possible input, and edge cases in how data is interpreted lead to unintended behaviour. The Process-ID (PID) is also relevant here -- it determines which privileges the running process already holds before any exploitation occurs.

### Privileges

Privileges control what the process is permitted to do on the system. They can be broken down into:

- **System** -- the highest level, referred to as SYSTEM on Windows and root on Linux, permitting any modification
- **User** -- permissions assigned to a specific account, often a dedicated service account on Linux
- **Groups** -- categorised sets of users sharing a common permission boundary
- **Policies** -- application-specific command execution rules applied to users or groups
- **Rules** -- granular permissions enforced from within the application itself

The privilege level of the process determines the blast radius of a successful exploitation. A vulnerability in a process running as SYSTEM or root is categorically more dangerous than the same vulnerability in a process running as a low-privileged service account.

### Destination

The Destination is where the result of the process ends up. This is either local -- changes to files, records, or other local services on the same system -- or network-based -- data forwarded to a remote address. Completing the Destination closes the cycle, and in many attack chains the Destination of one cycle becomes the Source of the next.

## Log4j as a Case Study

<img width="974" height="653" alt="image" src="https://github.com/user-attachments/assets/f7a68b48-cb6d-42f6-a622-b08ce85d3d6c" />

The [Log4j vulnerability (CVE-2021-44228)](https://cve.mitre.org/cgi-bin/cvename.cgi?name=cve-2021-44228) published in late 2021 is one of the clearest demonstrations of this model in action. Log4j is a widely used Java logging library integrated into a large number of commercial and open-source products. The attack unfolds across two complete cycles.

### Cycle 1 -- Initiating the Attack

| Step | What Happens | Category |
|---|---|---|
| 1 | Attacker manipulates the HTTP User-Agent header with a JNDI lookup command | Source |
| 2 | Log4j misinterprets the string as a command rather than text to be logged | Process |
| 3 | The JNDI lookup executes with administrator privileges due to logging permissions | Privileges |
| 4 | The lookup reaches out to an attacker-controlled server hosting a malicious Java class | Destination |

### Cycle 2 -- Triggering Remote Code Execution

| Step | What Happens | Category |
|---|---|---|
| 5 | The malicious Java class retrieved from the attacker's server becomes the new input | Source |
| 6 | The malicious code within the class is read and interpreted by the process | Process |
| 7 | Execution occurs under administrator privileges inherited from the logging process | Privileges |
| 8 | The code establishes a connection back to the attacker, granting remote system control | Destination |

GovCERT.ch published a detailed graphical breakdown of this exact flow at [https://www.govcert.ch/blog/zero-day-exploit-targeting-popular-java-library-log4j/](https://www.govcert.ch/blog/zero-day-exploit-targeting-popular-java-library-log4j/).

## Applying the Pattern

What made Log4j so severe was not just the code flaw -- it was the combination of a user-controlled Source (the User-Agent header), a Process that misinterpreted data as commands, Privileges that ran at administrator level because logging was considered sensitive, and a Destination that reached out to an external attacker-controlled server. Any one of those factors in isolation would have been far less dangerous.

This pattern template is not static. It grows more useful as it is applied to more services, more vulnerabilities, and more attack chains. Whether analysing a published CVE, reviewing source code for logic flaws, or debugging a custom exploit during development, mapping each step to Source, Process, Privileges, and Destination consistently reveals where the actual risk lies and why a given attack succeeds.
