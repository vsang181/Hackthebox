# Latest SMB Vulnerabilities

This section applies the Concept of Attacks framework to [SMBGhost (CVE-2020-0796)](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2020-0796), a critical vulnerability in the SMBv3.1.1 compression mechanism that affected Windows 10 versions 1903 and 1909. It allowed an unauthenticated attacker to achieve full remote code execution without any prior credentials or user interaction.

## The Vulnerability

SMBGhost is an [integer overflow](https://en.wikipedia.org/wiki/Integer_overflow) vulnerability in the SMB driver's compression handling function. An integer overflow occurs when an arithmetic operation produces a value that exceeds the maximum size of the memory space allocated for that variable. In this case, when the result is too large to fit, the excess data spills into adjacent memory -- overwriting whatever instructions were stored there.

The specific cause was a missing bounds check during SMB session negotiation. When a client and server negotiate an SMB connection, they agree on communication parameters including compression support. The function responsible for processing compressed packets did not validate whether the incoming data size exceeded what the allocated buffer could hold. An attacker could send a crafted compressed message that deliberately triggered this overflow, writing their own instructions into memory in place of the legitimate CPU instructions that would have executed next.

This is distinct from a simple misconfiguration or weak credential issue. The vulnerability exists in the logic of the driver itself, and triggering it requires no authentication -- only network access to TCP port 445.

## Mapping to the Concept of Attacks

As with Log4j and the CoreFTP vulnerability covered earlier, SMBGhost runs through two complete cycles.

### Cycle 1 -- Initiating the Attack

| Step | What Happens | Category |
|---|---|---|
| 1 | The attacker sends a specially crafted compressed SMB request to the target on TCP/445 | Source |
| 2 | The SMB driver processes the malformed compressed packets according to the negotiated protocol responses | Process |
| 3 | The SMB driver runs with system-level or administrator-level privileges | Privileges |
| 4 | The local SMB process becomes the destination, receiving and attempting to handle the malformed data | Destination |

### Cycle 2 -- Triggering Remote Code Execution

| Step | What Happens | Category |
|---|---|---|
| 5 | The overwritten buffer from cycle 1 becomes the source -- the attacker's instructions now reside in memory | Source |
| 6 | The integer overflow causes the CPU to execute the attacker's injected instructions rather than the legitimate driver code | Process |
| 7 | Execution continues under the same system-level privileges inherited from the SMB driver | Privileges |
| 8 | The attacker's machine becomes the destination -- a connection is established back, granting remote access to the compromised system | Destination |

A proof of concept is available on [Exploit-DB](https://www.exploit-db.com/exploits/48537), though fully weaponising it requires deeper knowledge of CPU internals, kernel memory layout, and exploit development techniques covered in the [Stack-Based Buffer Overflows on Linux x86](https://academy.hackthebox.com/course/preview/stack-based-buffer-overflows-on-linux-x86) and [Stack-Based Buffer Overflows on Windows x86](https://academy.hackthebox.com/course/preview/stack-based-buffer-overflows-on-windows-x86) modules.

## The Broader Lesson

What makes SMBGhost a useful case study alongside Log4j and CoreFTP is that it demonstrates the framework holds even for significantly more technical vulnerabilities. The exploit mechanism -- memory corruption through an integer overflow -- is far more complex than a path traversal or a JNDI injection. But the pattern is identical: an attacker-controlled input reaches a process, that process runs under elevated privileges, and the output reaches a destination that gives the attacker control.

The three vulnerabilities covered so far also illustrate how the Privileges category is often the deciding factor in impact. Log4j ran as an administrator because logs were considered sensitive. The CoreFTP write landed anywhere on the filesystem because permission checks were bypassed. SMBGhost executed with system privileges because that is what the SMB driver requires to function. In each case, the vulnerability was made significantly more dangerous not just by the flaw itself but by the privilege level attached to the process that contained it.
