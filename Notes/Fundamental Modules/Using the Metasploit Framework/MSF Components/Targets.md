# Targets

Targets in Metasploit are unique identifiers embedded within an exploit module that adapt the exploit's behaviour, specifically its return address and memory layout calculations, to a particular operating system version, service pack, language pack, or application version. A single exploit module can carry multiple targets, each with a different return address, because the same vulnerability may produce different memory layouts across different system configurations. The `show targets` command is context-dependent: issuing it from the root `msf6 >` prompt without a module loaded returns an error, while issuing it from within a loaded exploit module displays every target the module supports.

```bash
msf6 > show targets
[-] No exploit module selected.

msf6 exploit(windows/smb/ms17_010_psexec) > show targets

Exploit targets:
   Id  Name
   --  ----
   0   Automatic
```

## Why Targets Differ

The return address is the primary differentiating factor between targets. It points the processor to shellcode or a ROP gadget after the vulnerability is triggered, and that address is specific to the binary version loaded on the target system. Three common causes of variation between targets are:

- **Language packs:** A localised version of an OS or application loads DLLs at different base addresses, shifting all usable gadget addresses relative to the English language equivalent.
- **Service pack and patch level:** Updates often recompile binaries or reorder functions, invalidating any hardcoded return address from a prior version.
- **Hooks and third-party software:** Security products and runtime frameworks can shift module base addresses through hooks or by inserting additional code into the process address space.

The three most common return address types used in Metasploit targets are `jmp esp`, a jump to a specific register that points into attacker-controlled memory, and `pop/pop/ret`, which is used in structured exception handler (SEH) based exploits to redirect execution through the exception handler chain.

## Targets with Multiple Versions: IE execCommand UAF

The MS12-063 Internet Explorer `execCommand` use-after-free vulnerability provides a clear example of a module that requires specific target selection. The `info` command reveals the full picture before any execution is attempted, covering the vulnerability mechanism, ROP chain dependencies, and available targets:

```bash
msf6 exploit(windows/browser/ie_execcommand_uaf) > info

       Name: MS12-063 Microsoft Internet Explorer execCommand Use-After-Free Vulnerability
     Module: exploit/windows/browser/ie_execcommand_uaf
   Platform: Windows
       Rank: Good
  Disclosed: 2012-09-14

Available targets:
  Id  Name
  --  ----
  0   Automatic
  1   IE 7 on Windows XP SP3
  2   IE 8 on Windows XP SP3
  3   IE 7 on Windows Vista
  4   IE 8 on Windows Vista
  5   IE 8 on Windows 7
  6   IE 9 on Windows 7
```

The description clarifies important ROP chain dependencies that affect target selection: `msvcrt.dll` must be present for IE8 on Windows XP SP3 (it is by default), while IE8 or IE9 on Vista or Windows 7 require JRE 1.6.x or below to be installed for the ROP chain to function. This level of detail makes `info` an essential first step before using any unfamiliar module.

## Selecting a Target

Leaving the target set to `Automatic` (index 0) instructs the module to perform its own service detection against the target and select the most appropriate target entry from the list. This is reliable when the module includes robust fingerprinting logic, but it is not guaranteed to succeed in every case.

When the target OS, service pack, and application version are already known from enumeration, specify the target index directly for a more reliable and predictable execution:

```bash
msf6 exploit(windows/browser/ie_execcommand_uaf) > show targets

Exploit targets:
   Id  Name
   --  ----
   0   Automatic
   1   IE 7 on Windows XP SP3
   2   IE 8 on Windows XP SP3
   3   IE 7 on Windows Vista
   4   IE 8 on Windows Vista
   5   IE 8 on Windows 7
   6   IE 9 on Windows 7

msf6 exploit(windows/browser/ie_execcommand_uaf) > set target 6
target => 6
```

## Adding a Custom Target

When an exploit module does not include an entry for the specific version running on a target, a new target can be added manually. The process requires three steps:

1. Determine the type of return address required, using existing targets in the module's source code as a reference. Comments in the exploit Ruby file typically name the DLL and the address type in use.
2. Obtain a copy of the target binary or DLL from a matching system.
3. Use `msfpescan` to locate a usable return address within that binary:

```bash
# Scan for pop/pop/ret sequences
msfpescan -p umpnpmgr.dll

[umpnpmgr.dll]
0x79001567 pop eax; pop esi; ret
0x79012749 pop esi; pop ebp; retn 0x0010
0x7901285c pop edi; pop esi; retn 0x0004

# Disassemble at a known address to confirm the return type
msfpescan -D -a 0x767a38f6 umpnpmgr.dll
```

Once a suitable address is identified, it is added as a new target entry in the exploit module's `Targets` array, referencing the DLL it was sourced from in a comment for future maintainability. Target development and exploit customisation are covered in greater depth in the dedicated exploit development sections of this module.
